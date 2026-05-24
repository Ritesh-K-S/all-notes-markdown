# Chapter 2.6: Single Page Applications (SPA) vs Multi Page Applications (MPA)

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: The two fundamental approaches to building web app frontends — when pages fully reload (MPA) vs when only parts update dynamically (SPA), their trade-offs, and when to use each.

---

## 🧠 Real-Life Analogy: Book vs Flipbook

**MPA** = Reading a book. Every time you turn a page, you see a COMPLETELY new page. The old page is gone.

**SPA** = A magic book. The page stays the same, but words and pictures CHANGE on the page without turning it. Smooth and fast!

```
    MPA (Multi Page Application):
    ═════════════════════════════
    
    Click "About" link:
    
    ┌──────────┐                    ┌──────────┐
    │  Home    │   Full page       │  About   │
    │  Page    │   reload!         │  Page    │
    │          │   ┌──────────┐    │          │
    │  (HTML,  │──▶│ Browser  │───▶│  (NEW    │
    │   CSS,   │   │ goes     │    │   HTML,  │
    │   JS     │   │ white/   │    │   CSS,   │
    │   loaded)│   │ blank    │    │   JS     │
    └──────────┘   └──────────┘    │   loaded)│
                   "flash" ⚡      └──────────┘
    
    Server returns a BRAND NEW HTML page each time.
    
    
    SPA (Single Page Application):
    ══════════════════════════════
    
    Click "About" link:
    
    ┌──────────────────────────────────────┐
    │  Same page shell stays!              │
    │  ┌────────────────────────────────┐  │
    │  │  Header (stays)               │  │
    │  ├────────────────────────────────┤  │
    │  │                               │  │
    │  │  Only THIS part changes! ✨    │  │
    │  │                               │  │
    │  │  Home content → About content │  │
    │  │  (smooth transition)          │  │
    │  │                               │  │
    │  ├────────────────────────────────┤  │
    │  │  Footer (stays)               │  │
    │  └────────────────────────────────┘  │
    │                                      │
    │  URL changes: /home → /about         │
    │  But NO full page reload!            │
    └──────────────────────────────────────┘
    
    JavaScript fetches data and updates the DOM.
```

---

## 📖 Multi Page Application (MPA) — Deep Dive

### How MPA Works

```
    Every navigation = full round trip to the server
    ═════════════════════════════════════════════════
    
    User clicks "Products" link:
    
    Browser                                Server
       │                                      │
       │  GET /products                       │
       │ ────────────────────────────────────▶│
       │                                      │
       │  ┌─── Browser goes blank ───┐       │  Server:
       │  │  Old page destroyed      │       │  1. Query database
       │  │  CSS/JS unloaded         │       │  2. Render template
       │  │  White screen (flash!)   │       │  3. Return full HTML
       │  └──────────────────────────┘       │
       │                                      │
       │  200 OK                              │
       │  Content-Type: text/html             │
       │  Body: <!DOCTYPE html>               │
       │        <html>                        │
       │          <head>...</head>             │
       │          <body>                       │
       │            <nav>...</nav>             │
       │            <div>Products...</div>     │
       │            <footer>...</footer>       │
       │          </body>                      │
       │        </html>                        │
       │◀─────────────────────────────────────│
       │                                      │
       │  Browser: Parse HTML, load CSS,      │
       │  load JS, render page FROM SCRATCH   │
```

### MPA Example — Python (Flask)

```python
from flask import Flask, render_template
app = Flask(__name__)

# Each route returns a FULL HTML page
@app.route('/')
def home():
    return render_template('home.html')  # Full page

@app.route('/about')
def about():
    return render_template('about.html')  # Full page

@app.route('/products')
def products():
    items = db.query("SELECT * FROM products")
    return render_template('products.html', items=items)  # Full page

@app.route('/products/<int:id>')
def product_detail(id):
    item = db.query("SELECT * FROM products WHERE id=?", id)
    return render_template('product_detail.html', item=item)  # Full page
```

```html
<!-- templates/base.html — Shared layout (duplicated in every response!) -->
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet" href="/css/main.css">  <!-- Re-downloaded? -->
</head>
<body>
    <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>         <!-- Each click = full reload -->
        <a href="/products">Products</a>
    </nav>
    
    {% block content %}{% endblock %}
    
    <footer>© 2024 MyStore</footer>
    <script src="/js/main.js"></script>    <!-- Re-executed! -->
</body>
</html>
```

### MPA Example — Java (Spring Boot)

```java
@Controller
public class PageController {

    // Each method returns a FULL HTML page (via Thymeleaf template)
    @GetMapping("/")
    public String home(Model model) {
        return "home";  // → templates/home.html (full page)
    }

    @GetMapping("/products")
    public String products(Model model) {
        model.addAttribute("items", productRepo.findAll());
        return "products";  // → templates/products.html (full page)
    }

    @GetMapping("/products/{id}")
    public String productDetail(@PathVariable Long id, Model model) {
        model.addAttribute("item", productRepo.findById(id));
        return "product-detail";  // Full page reload for each product!
    }
}
```

---

## 📖 Single Page Application (SPA) — Deep Dive

### How SPA Works

```
    FIRST LOAD: Browser downloads the entire app
    ═════════════════════════════════════════════
    
    Browser                                Server
       │                                      │
       │  GET /                               │
       │ ────────────────────────────────────▶│
       │                                      │
       │  200 OK                              │
       │  Body: <html>                        │
       │          <div id="root"></div>        │  ← Nearly empty HTML!
       │          <script src="app.js">       │  ← The ENTIRE app
       │        </html>                        │
       │◀─────────────────────────────────────│
       │                                      │
       │  GET /app.js (1-5 MB!)               │
       │ ────────────────────────────────────▶│
       │  200 OK (huge JavaScript bundle)     │
       │◀─────────────────────────────────────│
       │                                      │
       │  JavaScript takes over:              │
       │  1. Creates the DOM                  │
       │  2. Sets up routing                  │
       │  3. Renders Home page                │
       │  4. Fetches data via API             │
    
    
    SUBSEQUENT NAVIGATION: Only data changes
    ═════════════════════════════════════════
    
    User clicks "Products":
    
    Browser                                API Server
       │                                      │
       │  URL changes: / → /products          │
       │  (via History API — NO reload!)      │
       │                                      │
       │  JavaScript router:                  │
       │  "URL = /products → render           │
       │   ProductList component"             │
       │                                      │
       │  GET /api/products                   │  ← Only fetch DATA
       │ ────────────────────────────────────▶│     (JSON, not HTML!)
       │                                      │
       │  200 OK                              │
       │  Body: [{"id":1,"name":"Phone"},     │
       │         {"id":2,"name":"Laptop"}]    │
       │◀─────────────────────────────────────│
       │                                      │
       │  JavaScript updates ONLY the         │
       │  content area. Header, footer,       │
       │  sidebar stay intact! ✨              │
       │                                      │
       │  No page reload! No white flash!     │
```

### SPA — Client-Side Routing

```
    How the URL changes WITHOUT a page reload:
    ═══════════════════════════════════════════
    
    Browser History API:
    
    // When user clicks "Products" link:
    history.pushState({}, '', '/products');
    
    // URL bar shows: mystore.com/products
    // But NO request to server!
    // JavaScript handles the "navigation"
    
    
    SPA Router (simplified):
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │  URL: /              → render <Home />          │
    │  URL: /products      → render <ProductList />   │
    │  URL: /products/42   → render <ProductDetail /> │
    │  URL: /about         → render <About />         │
    │  URL: /cart           → render <Cart />          │
    │  URL: * (anything)   → render <NotFound />      │
    │                                                 │
    └─────────────────────────────────────────────────┘
    
    The router watches for URL changes and swaps
    components WITHOUT reloading the page.
```

### SPA — React Example

```javascript
// App.js — React SPA with client-side routing
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function App() {
    return (
        <BrowserRouter>
            {/* This shell NEVER reloads */}
            <nav>
                <Link to="/">Home</Link>           {/* No page reload! */}
                <Link to="/products">Products</Link>
                <Link to="/about">About</Link>
            </nav>

            {/* Only this part changes */}
            <main>
                <Routes>
                    <Route path="/" element={<Home />} />
                    <Route path="/products" element={<ProductList />} />
                    <Route path="/products/:id" element={<ProductDetail />} />
                    <Route path="/about" element={<About />} />
                </Routes>
            </main>

            <footer>© 2024 MyStore</footer>  {/* Never reloads */}
        </BrowserRouter>
    );
}

// ProductList fetches data via API
function ProductList() {
    const [products, setProducts] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetch('/api/products')          // Only fetch JSON data!
            .then(res => res.json())
            .then(data => {
                setProducts(data);
                setLoading(false);
            });
    }, []);

    if (loading) return <p>Loading...</p>;

    return (
        <div>
            <h1>Products</h1>
            {products.map(p => (
                <Link to={`/products/${p.id}`} key={p.id}>
                    <div className="product-card">
                        <h2>{p.name}</h2>
                        <p>₹{p.price}</p>
                    </div>
                </Link>
            ))}
        </div>
    );
}
```

### SPA Backend — API Server (Python)

```python
from flask import Flask, jsonify
from flask_cors import CORS

app = Flask(__name__)
CORS(app)  # Allow frontend to call this API

# SPA backend serves JSON only — no HTML rendering!
@app.route('/api/products')
def get_products():
    products = db.query("SELECT * FROM products")
    return jsonify(products)  # JSON, not HTML!

@app.route('/api/products/<int:id>')
def get_product(id):
    product = db.query("SELECT * FROM products WHERE id=?", id)
    return jsonify(product)

# The SPA HTML is served by a static file server (Nginx, CDN, etc.)
# NOT by this API server!
```

---

## 📊 SPA vs MPA — Complete Comparison

```
    ┌────────────────────┬────────────────────────┬────────────────────────┐
    │  Aspect            │  MPA                   │  SPA                   │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Navigation        │  Full page reload      │  Dynamic update        │
    │                    │  (white flash)         │  (smooth, instant)     │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  First Load        │  ⚡ Fast (small HTML)  │  🐌 Slow (big JS)     │
    │                    │  Server sends ready    │  Must download &       │
    │                    │  HTML                  │  execute all JS        │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Subsequent        │  🐌 Slow (full reload  │  ⚡ Fast (only fetch   │
    │  Navigation        │  every time)           │  JSON data)            │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  SEO               │  ✅ Excellent          │  ❌ Poor (without SSR) │
    │                    │  Server-rendered HTML  │  Empty HTML initially  │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Server Load       │  🔴 Higher             │  💚 Lower              │
    │                    │  Renders HTML for      │  Only serves JSON      │
    │                    │  every request         │  (frontend does work)  │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  User Experience   │  🟡 Average            │  ✅ App-like           │
    │                    │  (page flashes)        │  (smooth transitions)  │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Offline Support   │  ❌ No                 │  ✅ Yes (w/ SW)        │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Complexity        │  💚 Simpler            │  🔴 More complex       │
    │                    │  Traditional approach  │  State management,     │
    │                    │                        │  routing, API design   │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Back/Forward      │  ✅ Works naturally    │  🟡 Must implement     │
    │  Button            │                        │  with History API      │
    ├────────────────────┼────────────────────────┼────────────────────────┤
    │  Frameworks        │  Django, Rails, Spring │  React, Angular, Vue   │
    │                    │  Flask, PHP, ASP.NET   │  Svelte, Next.js       │
    └────────────────────┴────────────────────────┴────────────────────────┘
```

---

## 🔄 The Performance Timeline

```
    FIRST PAGE LOAD:
    ═══════════════
    
    MPA:  ████████░░░░░░░░░░░░  Page visible in 800ms
                                (Server renders HTML, browser shows it)
    
    SPA:  ░░░░░░░░░░████████████████  Page visible in 2500ms
          ^^^^^^^^^^                  
          Downloading   Parsing JS    Fetching data    Rendering
          app.js        & executing   from API         to DOM
    
    
    SECOND PAGE (navigation):
    ═════════════════════════
    
    MPA:  ████████░░░░░░░░░░░░  Full reload: 800ms again
                                (everything from scratch!)
    
    SPA:  ██░░░░                Only data fetch: 200ms
          ^^                    (JS already loaded, just swap content)
          API call
    
    
    OVERALL USER SESSION (5 page views):
    ═════════════════════════════════════
    
    MPA:  800 + 800 + 800 + 800 + 800 = 4000ms total wait
    SPA:  2500 + 200 + 200 + 200 + 200 = 3300ms total wait
                                         (but first load was slow!)
```

---

## 🌐 Modern Approach: Hybrid (MPA + SPA Features)

Most modern frameworks blend both approaches:

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  HYBRID APPROACH (Best of Both Worlds):                      │
    │  ═════════════════════════════════════                        │
    │                                                              │
    │  First Load: Server renders full HTML (like MPA)             │
    │  → Fast initial paint, good SEO ✅                           │
    │                                                              │
    │  After Load: JavaScript takes over (like SPA)                │
    │  → Smooth navigation, no full reloads ✅                     │
    │                                                              │
    │  This is called HYDRATION:                                   │
    │                                                              │
    │  Server HTML ──▶ Browser shows page ──▶ JS "hydrates"       │
    │  (fast paint)     immediately            (attaches events,   │
    │                                          becomes interactive)│
    │                                                              │
    │  Frameworks that do this:                                    │
    │  • Next.js (React)                                           │
    │  • Nuxt.js (Vue)                                             │
    │  • SvelteKit (Svelte)                                        │
    │  • Angular Universal                                         │
    │  • Astro (Islands architecture)                              │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
    
    
    ASTRO "ISLANDS" — New Approach:
    ═══════════════════════════════
    
    ┌─────────────────────────────────────────────┐
    │  Static HTML page (MPA — no JS!)            │
    │                                             │
    │  ┌──────────────────────────────────────┐   │
    │  │  Header (static HTML, zero JS)      │   │
    │  └──────────────────────────────────────┘   │
    │                                             │
    │  ┌───────────┐   ┌──────────────────────┐   │
    │  │  Static   │   │  Interactive         │   │
    │  │  Content  │   │  "Island" 🏝️         │   │
    │  │  (no JS)  │   │  (React component   │   │
    │  │           │   │   with JS — search   │   │
    │  │           │   │   bar, cart, etc.)   │   │
    │  └───────────┘   └──────────────────────┘   │
    │                                             │
    │  ┌──────────────────────────────────────┐   │
    │  │  Footer (static HTML, zero JS)      │   │
    │  └──────────────────────────────────────┘   │
    │                                             │
    │  Only the "islands" load JavaScript.        │
    │  Rest of the page = pure HTML.              │
    │  Result: MPA speed + SPA interactivity!     │
    └─────────────────────────────────────────────┘
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬───────────┬──────────────────────────────────┐
    │  Company         │  Type     │  Why                             │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Gmail           │  SPA      │  Email is interactive — drafts,  │
    │                  │           │  threads, real-time updates      │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Google Maps     │  SPA      │  Map interactions need smooth    │
    │                  │           │  rendering without page reloads  │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Figma           │  SPA      │  Complex canvas app, loads once  │
    │                  │           │  and runs like a desktop app     │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Wikipedia       │  MPA      │  SEO is critical. Each article   │
    │                  │           │  is a separate page.             │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Amazon          │  MPA      │  SEO for product pages. Server   │
    │                  │  (hybrid) │  renders, then JS enhances       │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Netflix         │  Hybrid   │  SSR for landing page (SEO),     │
    │                  │           │  SPA after login (smooth UX)     │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Shopify stores  │  MPA      │  Each product page = separate    │
    │                  │  + SSG    │  static page (SEO + speed)       │
    ├──────────────────┼───────────┼──────────────────────────────────┤
    │  Twitter/X       │  SPA      │  Infinite scroll, real-time      │
    │                  │           │  updates, app-like experience    │
    └──────────────────┴───────────┴──────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Building a blog/docs site as SPA
       → Massive JS bundle for simple static content!
       ✅ Use SSG (MPA) for content-heavy sites

    ❌ Building a complex dashboard as MPA
       → Full page reload for every filter/sort = terrible UX
       ✅ Use SPA for highly interactive applications

    ❌ Forgetting SEO in SPA
       → Google may not index JavaScript-rendered content
       ✅ Use SSR/SSG framework (Next.js, Nuxt.js) for public pages

    ❌ Giant SPA bundles without code splitting
       → Users download 5MB of JS for a page that needs 200KB
       ✅ Use dynamic imports: React.lazy(), route-based splitting

    ❌ Not handling browser back/forward in SPA
       → Back button doesn't work or breaks state
       ✅ Use proper router (react-router, vue-router)

    ❌ Not pre-rendering critical pages
       → First visit = blank screen while JS loads
       ✅ Use SSR for first paint, then hydrate to SPA
```

---

## ✅ Decision Guide: SPA or MPA?

```
    Choose MPA if:                       Choose SPA if:
    ══════════════                       ══════════════
    ✅ SEO is critical                   ✅ Complex interactivity
    ✅ Content-focused site              ✅ Real-time updates needed
    ✅ Blog, docs, marketing             ✅ Dashboard, admin panel
    ✅ Simple team / less JS expertise   ✅ Desktop-app-like UX needed
    ✅ Need accessibility by default     ✅ Offline support needed
    ✅ Low-bandwidth users               ✅ Team has JS expertise
    
    Choose HYBRID if:
    ═════════════════
    ✅ Need both SEO AND interactivity (most modern apps!)
    ✅ E-commerce (SSG product pages + SPA cart)
    ✅ Social media (SSR for public profiles + SPA for feed)
    ✅ News sites (SSG articles + SPA comments)
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. MPA = each navigation loads a NEW HTML page from the server.    ║
║     Simple, great SEO, but slow navigation (full page reloads).     ║
║                                                                      ║
║  2. SPA = one HTML page + JavaScript handles all navigation.        ║
║     Smooth UX, but slower first load and poor SEO without SSR.      ║
║                                                                      ║
║  3. First load: MPA wins. Subsequent navigation: SPA wins.         ║
║     For best of both: use Hybrid (SSR + hydration).                 ║
║                                                                      ║
║  4. Modern frameworks (Next.js, Nuxt, SvelteKit) blur the line —  ║
║     they server-render first, then hydrate into SPA behavior.       ║
║                                                                      ║
║  5. Islands architecture (Astro) = MPA with tiny interactive       ║
║     "islands" that use JS. Minimal JS for maximum speed.            ║
║                                                                      ║
║  6. RULE OF THUMB: Content sites → MPA/SSG. Interactive apps →     ║
║     SPA. Most real-world apps → Hybrid.                             ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

As SPAs grow bigger, they become hard to maintain with one team owning the entire frontend. In [Chapter 2.7: Micro-Frontends](./07-micro-frontends.md), we'll learn how companies like Amazon and IKEA break their frontend into independent pieces owned by different teams.

---

[⬅️ Previous: Frontend Caching](./05-frontend-caching.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Micro-Frontends ➡️](./07-micro-frontends.md)
