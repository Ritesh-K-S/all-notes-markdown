# Chapter 2.1: How the Browser Renders a Web Page (DOM, CSSOM, Paint)

> **Level**: ⭐ Beginner  
> **What you'll learn**: The complete step-by-step process your browser follows to transform raw HTML, CSS, and JavaScript into the beautiful page you see on screen.

---

## 🧠 Real-Life Analogy: Building a House

Imagine you're building a house. You don't just dump bricks, paint, and furniture into a pile and call it home. You follow a process:

```
    BUILDING A HOUSE                      RENDERING A WEB PAGE
    ════════════════                      ══════════════════════

    1. Read blueprints        ──▶        1. Parse HTML
       (structure plan)                     (build DOM tree)

    2. Read interior design   ──▶        2. Parse CSS
       (colors, materials)                  (build CSSOM tree)

    3. Combine both plans     ──▶        3. Combine DOM + CSSOM
       (what goes where,                    (build Render Tree)
        what it looks like)

    4. Measure & position     ──▶        4. Layout
       every room & furniture               (calculate positions & sizes)

    5. Actually build it      ──▶        5. Paint
       (lay bricks, paint walls)            (draw pixels on screen)

    6. Final touches          ──▶        6. Composite
       (glass, frames, layers)              (layer management)
```

Your browser does this **every single time** you load a web page — and it does it in **milliseconds**.

---

## 📖 The Rendering Pipeline — Overview

When your browser receives an HTML file, it goes through this exact pipeline:

```
    HTML File                CSS Files              JavaScript Files
       │                        │                        │
       ▼                        ▼                        │
    ┌──────────┐         ┌──────────┐                    │
    │  HTML    │         │  CSS     │                    │
    │  Parser  │         │  Parser  │                    │
    └────┬─────┘         └────┬─────┘                    │
         │                    │                          │
         ▼                    ▼                          │
    ┌──────────┐         ┌──────────┐                    │
    │   DOM    │         │  CSSOM   │                    │
    │   Tree   │         │  Tree    │                    │
    └────┬─────┘         └────┬─────┘                    │
         │                    │                          ▼
         └────────┬───────────┘                   ┌──────────┐
                  │                               │    JS    │
                  ▼                               │  Engine  │
           ┌──────────┐                           │(Can modify│
           │  Render  │◀──────────────────────────│ DOM/CSSOM)│
           │   Tree   │                           └──────────┘
           └────┬─────┘
                │
                ▼
           ┌──────────┐
           │  Layout  │   "Where does each element go?"
           │ (Reflow) │   "How big is it?"
           └────┬─────┘
                │
                ▼
           ┌──────────┐
           │  Paint   │   "What color? What border?"
           │          │   "Draw the actual pixels"
           └────┬─────┘
                │
                ▼
           ┌──────────┐
           │Composite │   "Layer management for"
           │          │   "smooth animations"
           └────┬─────┘
                │
                ▼
           ╔══════════╗
           ║  PIXELS  ║   What you see on screen! 🎉
           ║ ON SCREEN║
           ╚══════════╝
```

Let's dive into each step.

---

## Step 1: HTML Parsing → Building the DOM Tree

When the browser receives HTML, it reads it character by character and builds a **DOM (Document Object Model)** tree — a structured representation of the page.

### What is the DOM?

The **DOM** is a tree-like data structure that represents every element in your HTML as a **node** (object). It's how the browser "understands" your HTML.

```
    HTML Source Code:
    ─────────────────
    <html>
      <head>
        <title>My Store</title>
        <link rel="stylesheet" href="styles.css">
      </head>
      <body>
        <header>
          <h1>Welcome to My Store</h1>
          <nav>
            <a href="/products">Products</a>
            <a href="/about">About</a>
          </nav>
        </header>
        <main>
          <p>Best products <strong>ever</strong>!</p>
          <img src="hero.jpg" alt="Hero">
        </main>
      </body>
    </html>


    DOM Tree (what the browser builds):
    ────────────────────────────────────

                          Document
                             │
                           html
                        ┌────┴────┐
                      head       body
                    ┌──┴──┐    ┌──┴──────┐
                 title   link header    main
                   │            │      ┌──┴──┐
               "My Store"    ┌──┴──┐  p     img
                            h1    nav  │
                             │   ┌─┴─┐ ├────────┐
                       "Welcome" a   a "Best.." strong
                                 │   │           │
                           "Products" "About"  "ever"
```

### How the HTML Parser Works — Token by Token

```
    HTML Source:  <html><head><title>Hi</title></head><body>...</body></html>
                   │
                   ▼
    TOKENIZER breaks it into tokens:
    
    Token 1: StartTag    <html>
    Token 2: StartTag    <head>
    Token 3: StartTag    <title>
    Token 4: Character   "Hi"
    Token 5: EndTag      </title>
    Token 6: EndTag      </head>
    Token 7: StartTag    <body>
    Token 8: Character   "..."
    Token 9: EndTag      </body>
    Token 10: EndTag     </html>
    
    TREE BUILDER uses these tokens to construct the DOM tree:
    
    Token 1 <html>   → Create html node as root
    Token 2 <head>   → Create head node, child of html
    Token 3 <title>  → Create title node, child of head
    Token 4 "Hi"     → Create text node, child of title
    Token 5 </title> → Go back up to head
    Token 6 </head>  → Go back up to html
    Token 7 <body>   → Create body node, child of html
    ... and so on
```

### Important: HTML Parsing is **Forgiving**

Unlike most programming languages, HTML parsers **don't crash** on errors:

```
    This broken HTML:                    Browser still builds a valid DOM:
    
    <p>Hello                             <html>
    <div>World                             <head></head>
    </p>                                   <body>
    <b>Bold<i>Both</b>Italic</i>            <p>Hello</p>
                                            <div>World</div>
                                            <b>Bold<i>Both</i></b>
                                            <i>Italic</i>
                                          </body>
                                         </html>
    
    The browser FIXES your mistakes automatically!
    (But don't rely on this — write proper HTML)
```

---

## Step 2: CSS Parsing → Building the CSSOM Tree

While the DOM is being built, the browser also downloads and parses **CSS files** to build the **CSSOM (CSS Object Model)** tree.

```
    CSS Source:
    ──────────
    body {
        font-family: Arial;
        font-size: 16px;
        margin: 0;
    }
    header {
        background: #333;
        color: white;
        padding: 20px;
    }
    h1 {
        font-size: 32px;
    }
    p {
        line-height: 1.6;
        color: #444;
    }
    strong {
        color: red;
    }


    CSSOM Tree:
    ───────────

                          Document
                             │
                     ┌───────┴───────┐
                   body              (user agent styles)
                   │ font: Arial
                   │ size: 16px
                   │ margin: 0
                   │
             ┌─────┼─────────┐
          header  main      ...
          │ bg: #333
          │ color: white
          │ padding: 20px
          │
       ┌──┴──┐
      h1    nav         p              strong
      │ size: 32px      │ line-height:  │ color: red
      │ color: white    │ 1.6           │
      │ (inherited)     │ color: #444   │
```

### CSS is Render-Blocking!

```
    ┌────────────────────────────────────────────────────────────┐
    │                     ⚠️ CRITICAL POINT                      │
    │                                                            │
    │  The browser STOPS rendering until ALL CSS is downloaded   │
    │  and parsed. Why? Because without CSS, the browser         │
    │  doesn't know what anything looks like.                    │
    │                                                            │
    │  If CSS loaded AFTER rendering:                            │
    │  1. Page appears unstyled (ugly flash)                     │
    │  2. Then jumps to styled version (bad user experience)     │
    │  This is called FOUC (Flash of Unstyled Content)           │
    │                                                            │
    │  Timeline:                                                 │
    │  HTML downloading: ████████████                            │
    │  CSS downloading:      ████████████  ← BLOCKS rendering   │
    │  Rendering:                         ████████████           │
    │                                     ↑                      │
    │                              Can't start until             │
    │                              CSS is fully parsed!          │
    └────────────────────────────────────────────────────────────┘
```

### Style Computation — Cascading, Specificity & Inheritance

The "C" in CSS stands for **Cascading** — rules have an order of priority:

```
    PRIORITY ORDER (lowest to highest):
    ═══════════════════════════════════
    
    1. Browser defaults       (user-agent stylesheet)
       │  p { margin: 16px 0 }
       │
       2. External stylesheet    (your .css files)
       │  p { margin: 10px 0; color: #333 }
       │
       3. Internal <style> tag   (in the HTML)
       │  p { color: blue }
       │
       4. Inline style           (on the element)
       │  <p style="color: red">
       │
       5. !important             (overrides everything — avoid!)
          p { color: green !important; }
    
    SPECIFICITY (which rule wins when they conflict):
    
    Inline style     → 1000 points   <p style="...">
    #id selector     → 100 points    #header { }
    .class selector  → 10 points     .nav-item { }
    element selector → 1 point       p { }
    
    Example:
    p { color: blue; }                    → 1 point
    .text { color: green; }               → 10 points    ← WINS!
    #main p.text { color: red; }          → 111 points   ← WINS OVER ABOVE!
    
    INHERITANCE (some styles pass from parent to child):
    
    body { color: #333; font-family: Arial; }
      │
      └── h1 inherits color #333 and font-family Arial
      └── p inherits color #333 and font-family Arial
          └── strong inherits font-family Arial (but has own color)
    
    Inherited:     color, font-*, text-*, line-height, visibility
    NOT inherited: margin, padding, border, background, width, height
```

---

## Step 3: Building the Render Tree

The browser combines the **DOM** and **CSSOM** into a **Render Tree** — containing ONLY the elements that will actually be **visible** on screen.

```
    DOM Tree                   CSSOM Tree                Render Tree
    ────────                   ──────────                ───────────
    
    html                       body{...}                 body
    ├── head                   header{...}               ├── header
    │   ├── title              h1{...}                   │   ├── h1
    │   ├── link               p{...}                    │   │   └── "Welcome..."
    │   └── script             strong{...}               │   └── nav
    ├── body                   .hidden{                  │       ├── a "Products"
    │   ├── header               display:none            │       └── a "About"
    │   │   ├── h1             }                         ├── main
    │   │   └── nav                                      │   ├── p
    │   ├── main                                         │   │   ├── "Best..."
    │   │   ├── p                                        │   │   └── strong "ever"
    │   │   └── img                                      │   └── img
    │   └── div.hidden                                   │
    │       └── "Secret"                                 (div.hidden is EXCLUDED!)
    │                                                    (head is EXCLUDED!)
    │                                                    (script is EXCLUDED!)
    
    
    What's EXCLUDED from the Render Tree:
    • <head>, <title>, <meta>, <link>, <script> — not visual
    • Elements with display: none — explicitly hidden
    • Elements inside <template> — not rendered
    
    What's INCLUDED but invisible:
    • Elements with visibility: hidden — take up space but invisible
    • Elements with opacity: 0 — transparent but take up space
```

---

## Step 4: Layout (Reflow) — Calculating Positions & Sizes

The browser walks through the Render Tree and calculates the **exact position and size** of every element on the page.

```
    ┌───────────────────── Viewport (1920 x 1080) ──────────────────┐
    │                                                                │
    │  ┌──── header ─────────────────────────────────────────────┐   │
    │  │  x: 0, y: 0                                             │   │
    │  │  width: 1920px, height: 80px                            │   │
    │  │                                                         │   │
    │  │  ┌── h1 ────────────────────────────────────────────┐   │   │
    │  │  │  x: 20, y: 20                                    │   │   │
    │  │  │  width: 400px, height: 40px                      │   │   │
    │  │  │  "Welcome to My Store"                           │   │   │
    │  │  └──────────────────────────────────────────────────┘   │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                │
    │  ┌──── main ───────────────────────────────────────────────┐   │
    │  │  x: 0, y: 80                                            │   │
    │  │  width: 1920px, height: 900px                           │   │
    │  │                                                         │   │
    │  │  ┌── p ─────────────────────────────────────────────┐   │   │
    │  │  │  x: 20, y: 100                                   │   │   │
    │  │  │  width: 1880px, height: 24px                     │   │   │
    │  │  │  "Best products ever!"                           │   │   │
    │  │  └──────────────────────────────────────────────────┘   │   │
    │  │                                                         │   │
    │  │  ┌── img ───────────────────┐                           │   │
    │  │  │  x: 20, y: 140          │                           │   │
    │  │  │  width: 800px           │                           │   │
    │  │  │  height: 400px          │                           │   │
    │  │  └─────────────────────────┘                           │   │
    │  └─────────────────────────────────────────────────────────┘   │
    │                                                                │
    └────────────────────────────────────────────────────────────────┘
    
    The browser uses the BOX MODEL for every element:
    
    ┌──────────────────────────────────────────────────┐
    │  MARGIN (space outside the border)               │
    │  ┌──────────────────────────────────────────┐    │
    │  │  BORDER (the visible edge)                │    │
    │  │  ┌──────────────────────────────────┐     │    │
    │  │  │  PADDING (space inside border)   │     │    │
    │  │  │  ┌──────────────────────────┐    │     │    │
    │  │  │  │                          │    │     │    │
    │  │  │  │       CONTENT            │    │     │    │
    │  │  │  │  (text, image, etc.)     │    │     │    │
    │  │  │  │                          │    │     │    │
    │  │  │  └──────────────────────────┘    │     │    │
    │  │  └──────────────────────────────────┘     │    │
    │  └──────────────────────────────────────────┘    │
    └──────────────────────────────────────────────────┘
    
    Total width = margin-left + border-left + padding-left 
                  + content-width 
                  + padding-right + border-right + margin-right
```

### Layout Is Expensive!

```
    ┌──────────────────────────────────────────────────────────────┐
    │  ⚠️ Layout can be triggered multiple times!                  │
    │                                                              │
    │  THINGS THAT TRIGGER LAYOUT (REFLOW):                       │
    │  • Resizing the browser window                              │
    │  • Changing font size                                       │
    │  • Adding/removing DOM elements                             │
    │  • Changing element dimensions (width, height, padding)     │
    │  • Reading certain properties: offsetWidth, clientHeight    │
    │  • Changing CSS: display, position, float                   │
    │                                                              │
    │  Every layout recalculates positions of MANY elements       │
    │  because elements affect each other's positions!            │
    │                                                              │
    │  A change to one element can cascade and affect hundreds.   │
    │  This is why frequent DOM manipulation is SLOW.             │
    └──────────────────────────────────────────────────────────────┘
```

---

## Step 5: Paint — Drawing the Pixels

Now the browser knows **where** everything goes. Time to actually **draw** it.

The paint step converts the layout into actual **pixels** — colors, text, borders, shadows, images.

```
    Paint creates PAINT RECORDS — instructions for what to draw:
    
    ┌─────────────────────────────────────────────────────┐
    │  Paint Record 1: Draw background #FFFFFF at (0,0)   │
    │  Paint Record 2: Draw rect #333 at (0,0,1920,80)    │ ← header bg
    │  Paint Record 3: Draw text "Welcome..." at (20,20)  │ ← h1 text
    │                  font: Arial 32px, color: white      │
    │  Paint Record 4: Draw text "Products" at (20,50)    │ ← nav link
    │                  font: Arial 16px, color: #0066cc    │
    │  Paint Record 5: Draw image hero.jpg at (20,140)    │ ← img
    │                  size: 800x400                        │
    │  Paint Record 6: Draw text "Best prod..." at (20,100)│ ← p text
    │  Paint Record 7: Draw text "ever" at (180,100)       │ ← strong
    │                  color: red, font-weight: bold        │
    │  ...                                                 │
    └─────────────────────────────────────────────────────┘
    
    These instructions are sent to the GPU for actual rendering.
```

### Paint Order (z-index & stacking context)

```
    Elements are painted in this order:
    
    1. Background colors
    2. Background images
    3. Borders
    4. Children (in DOM order)
    5. Outlines
    
    For OVERLAPPING elements (z-index):
    
    ┌──────────────────────────────┐
    │  z-index: 1 (painted first) │    ← BACK
    │  ┌────────────────────────┐ │
    │  │ z-index: 2             │ │
    │  │ ┌──────────────────┐   │ │
    │  │ │ z-index: 3       │   │ │    ← FRONT (painted last)
    │  │ │ (on top)         │   │ │
    │  │ └──────────────────┘   │ │
    │  └────────────────────────┘ │
    └──────────────────────────────┘
    
    Higher z-index = painted later = appears on top
```

---

## Step 6: Compositing — Layering for Performance

Modern browsers separate the page into **layers** that can be independently managed and animated by the **GPU**.

```
    Without Compositing:                With Compositing:
    ─────────────────────               ─────────────────────
    
    Every animation repaints           Layers are separate textures
    the ENTIRE page. Slow! 🐌          on the GPU. Fast! 🚀
    
    ┌──────────────────────┐           ┌─── Layer 1 (background) ──┐
    │                      │           │  ┌── Layer 2 (header) ──┐ │
    │  Everything is one   │           │  │ ┌ Layer 3 (menu) ──┐ │ │
    │  big flat painting   │           │  │ │  Animates         │ │ │
    │                      │           │  │ │  independently!   │ │ │
    │  Changing one pixel  │           │  │ └───────────────────┘ │ │
    │  repaints everything │           │  └───────────────────────┘ │
    │                      │           │                            │
    └──────────────────────┘           └────────────────────────────┘
    
    Things that create new layers:
    • position: fixed/sticky
    • transform / opacity animations
    • will-change: transform
    • <video>, <canvas>, <iframe>
    • CSS filters (blur, etc.)
    • z-index with certain positioning
```

---

## 🔄 How JavaScript Affects Rendering

JavaScript can **modify the DOM and CSSOM**, which re-triggers parts of the pipeline:

```
    ┌──────────────────────────────────────────────────────────────────┐
    │                                                                  │
    │  JAVASCRIPT IS PARSER-BLOCKING!                                 │
    │                                                                  │
    │  When the HTML parser encounters a <script> tag:                │
    │                                                                  │
    │  HTML Parsing:  ████████████ ── STOP ──█████████████████████    │
    │  JS Download:                ██████████                         │
    │  JS Execute:                           ████████                 │
    │  HTML Parsing:                                  ████████████    │
    │                                                                  │
    │  Why? Because JS might modify the DOM!                          │
    │  document.write() can completely change the HTML.               │
    │  So the parser MUST wait.                                       │
    │                                                                  │
    │  SOLUTIONS:                                                     │
    │                                                                  │
    │  1. <script defer src="app.js">                                 │
    │     Download in parallel, execute AFTER HTML is fully parsed    │
    │                                                                  │
    │     HTML Parsing:  ████████████████████████████████████████     │
    │     JS Download:         ██████████                             │
    │     JS Execute:                                    ████████    │
    │                                                                  │
    │  2. <script async src="analytics.js">                           │
    │     Download in parallel, execute as soon as downloaded         │
    │     (may run before HTML is done — use for independent scripts) │
    │                                                                  │
    │     HTML Parsing:  ██████████████ ─STOP─ ██████████████████    │
    │     JS Download:        ██████████                              │
    │     JS Execute:                   ████████                      │
    │                                                                  │
    └──────────────────────────────────────────────────────────────────┘

    BEST PRACTICE:
    <script defer src="app.js">        ← For your app code
    <script async src="analytics.js">  ← For third-party scripts
```

### What Triggers What? (The Cost of DOM Changes)

```
    JS Change                    Triggers          Cost
    ─────────                    ────────          ────

    element.style.color = 'red'  → Repaint only    💚 Low
                                   (no layout)

    element.style.fontSize='20px'→ Layout + Paint   🟡 Medium
                                   (size changed!)

    element.remove()             → Layout + Paint   🟡 Medium
                                   (everything shifts)

    element.style.transform      → Composite only   💚 Very Low
    = 'translateX(100px)'          (GPU handles it!)

    element.style.opacity = 0.5  → Composite only   💚 Very Low
                                   (GPU handles it!)

    element.offsetHeight         → FORCED Layout!    🔴 High
    (reading layout property)      (even if nothing
                                    visually changed)
    
    RULE OF THUMB:
    ═══════════════
    • Prefer transform/opacity for animations (GPU compositing)
    • Avoid changing width/height/top/left in animations (triggers layout)
    • Batch DOM reads and writes — don't interleave them!
```

---

## 💻 Code Examples — Observing the Rendering Process

### Python — Serving HTML for the Browser to Render
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    # The browser will receive this HTML and go through
    # the entire rendering pipeline we just learned about
    return '''
    <!DOCTYPE html>
    <html>
    <head>
        <style>
            /* CSSOM: These rules build the CSSOM tree */
            body { font-family: Arial; margin: 0; }
            .header { background: #2c3e50; color: white; padding: 20px; }
            .content { padding: 20px; }
            /* This creates a new compositing layer */
            .animated { will-change: transform; transition: transform 0.3s; }
            .animated:hover { transform: scale(1.1); }
        </style>
    </head>
    <body>
        <!-- DOM: Parser builds nodes from these elements -->
        <div class="header">
            <h1>My Store</h1>
        </div>
        <div class="content">
            <p>Welcome!</p>
            <button class="animated">Hover me (GPU animation!)</button>
        </div>
        <!-- defer: Downloads in parallel, runs after HTML parsed -->
        <script defer src="/static/app.js"></script>
    </body>
    </html>
    '''

if __name__ == '__main__':
    app.run(port=5000)
```

### Java (Spring Boot) — Serving HTML
```java
@Controller
public class HomeController {

    @GetMapping("/")
    public String home(Model model) {
        // Thymeleaf template — browser receives compiled HTML
        // and goes through: DOM → CSSOM → Render Tree → Layout → Paint
        model.addAttribute("title", "My Store");
        model.addAttribute("products", productService.getAll());
        return "home";  // renders templates/home.html
    }
}
```

### JavaScript — DOM Manipulation & Rendering Performance
```javascript
// BAD: Triggers layout 4 times (read-write-read-write pattern)
const box = document.getElementById('box');
const height = box.offsetHeight;     // READ → forces layout
box.style.height = height + 10 + 'px';  // WRITE → invalidates layout
const width = box.offsetWidth;       // READ → forces layout AGAIN!
box.style.width = width + 10 + 'px';    // WRITE → invalidates AGAIN!

// GOOD: Batch reads, then batch writes
const box = document.getElementById('box');
// Batch READS (one layout calculation)
const height = box.offsetHeight;
const width = box.offsetWidth;
// Batch WRITES (one layout recalculation)
box.style.height = height + 10 + 'px';
box.style.width = width + 10 + 'px';

// BEST: Use requestAnimationFrame for smooth animations
function animate() {
    element.style.transform = `translateX(${position}px)`;  // GPU only!
    position += 2;
    if (position < 500) {
        requestAnimationFrame(animate);  // Syncs with display refresh
    }
}
requestAnimationFrame(animate);
```

---

## 🏢 Real-World Example: How Google Optimizes Rendering

```
    Google Search Page Loading Strategy:
    ═════════════════════════════════════
    
    1. CRITICAL CSS is INLINED in <head> (no external CSS blocking!)
       Only ~14 KB of CSS needed for above-the-fold content
    
    2. JavaScript is deferred — search results render FIRST
       JS for autocomplete, voice search loads AFTER
    
    3. Images use lazy loading — only load when scrolled into view
    
    4. Fonts use font-display: swap — show text immediately with
       fallback font, swap to Google font when loaded
    
    5. Preconnect to critical origins:
       <link rel="preconnect" href="https://fonts.googleapis.com">
    
    Google's Core Web Vitals targets:
    ┌─────────────────────┬──────────────┐
    │  Metric             │  Target      │
    ├─────────────────────┼──────────────┤
    │  LCP (Largest       │  < 2.5s      │
    │  Contentful Paint)  │              │
    │  FID (First Input   │  < 100ms     │
    │  Delay)             │              │
    │  CLS (Cumulative    │  < 0.1       │
    │  Layout Shift)      │              │
    └─────────────────────┴──────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  ❌ Putting <link> CSS at the bottom of <body>               │
    │     → Causes Flash of Unstyled Content (FOUC)               │
    │     ✅ Always put CSS in <head>                              │
    │                                                              │
    │  ❌ Putting <script> in <head> without defer/async           │
    │     → Blocks HTML parsing, delays page display              │
    │     ✅ Use <script defer> or put scripts before </body>      │
    │                                                              │
    │  ❌ Animating width/height/top/left with JS                  │
    │     → Triggers expensive layout + paint on every frame      │
    │     ✅ Use transform and opacity (GPU composited)            │
    │                                                              │
    │  ❌ Reading layout props, then writing, in a loop            │
    │     → Causes "layout thrashing" — forced synchronous layout │
    │     ✅ Batch all reads first, then all writes                │
    │                                                              │
    │  ❌ Loading huge images that are displayed small              │
    │     → Wastes bandwidth and memory                           │
    │     ✅ Use responsive images: srcset, sizes attributes       │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## ✅ When to Care About Rendering Performance

```
    CARE A LOT:                          CARE LESS:
    • E-commerce (every 100ms delay      • Admin dashboards (internal)
      = 1% revenue drop — Amazon)        • Dev tools
    • Media sites (fast content)         • One-time use pages
    • Mobile users (slow devices)        • Print-only pages
    • SEO-critical pages
    • Landing pages (first impression)
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Rendering pipeline: HTML→DOM, CSS→CSSOM, DOM+CSSOM→Render Tree, ║
║     Render Tree→Layout→Paint→Composite→Pixels on screen             ║
║                                                                      ║
║  2. CSS is RENDER-BLOCKING — browser waits for all CSS before       ║
║     painting anything. Put CSS in <head>.                            ║
║                                                                      ║
║  3. JavaScript is PARSER-BLOCKING — use defer or async to avoid     ║
║     stopping HTML parsing.                                           ║
║                                                                      ║
║  4. Layout (reflow) is expensive — changing sizes/positions of      ║
║     elements triggers recalculation for many elements.               ║
║                                                                      ║
║  5. Use transform/opacity for animations — they only trigger         ║
║     compositing (GPU), not layout or paint.                          ║
║                                                                      ║
║  6. The Render Tree excludes invisible elements (display:none,      ║
║     <head>, <script>) but includes opacity:0 and visibility:hidden. ║
║                                                                      ║
║  7. Use browser DevTools (F12 → Performance tab) to see the         ║
║     rendering pipeline in action for any website.                    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

Now that you understand how the browser renders a page, the next question is: **where** does that HTML get generated? On the client? On the server? At build time? That's exactly what we cover in [Chapter 2.2: CSR vs SSR](./02-csr-vs-ssr.md).

---

[⬅️ Previous: Ports, Sockets & Connections](../01-basics/07-ports-sockets-connections.md) | [⬆️ Index](../../00-INDEX.md) | [Next: CSR vs SSR ➡️](./02-csr-vs-ssr.md)
