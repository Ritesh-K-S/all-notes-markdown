# Performance Budgets & Core Web Vitals

> **What you'll learn**: How to define, measure, and enforce performance targets for your web application — including Google's Core Web Vitals that directly affect search rankings and user experience.

---

## Real-Life Analogy

Imagine you're building a house with a **financial budget**:

- Before construction starts, you decide: "We can spend max $300K"
- As you build, you track spending: "Foundation: $50K, Framing: $80K, Plumbing: $40K..."
- If the electrician quote pushes you over budget, you either find a cheaper option or cut something else
- You don't just "hope" you stay within budget — you **enforce** it at every decision

A **performance budget** works the same way, but for web performance:

- "Our page must load in under 3 seconds"
- "Total JavaScript must stay under 300KB"
- "Largest image must be under 200KB"
- Every time a developer adds code or assets, the budget is checked automatically

```
Financial Budget:                    Performance Budget:
┌──────────────────────────┐        ┌──────────────────────────┐
│ Total: $300,000          │        │ Total Load Time: 3s      │
│                          │        │                          │
│ Foundation:  $50K  ██    │        │ HTML:        15KB  ▓     │
│ Framing:     $80K  ███   │        │ CSS:         50KB  ██    │
│ Plumbing:    $40K  ██    │        │ JavaScript: 250KB  █████ │
│ Electrical:  $60K  ██▓   │        │ Images:     400KB  ██████│
│ Finishings:  $70K  ███   │        │ Fonts:       80KB  ██    │
│ ─────────────────────    │        │ ─────────────────────    │
│ Total:      $300K  ✅    │        │ Total:      795KB  ✅    │
│ Over budget? BUILD STOPS │        │ Over budget? BUILD FAILS │
└──────────────────────────┘        └──────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is a Performance Budget?

A performance budget is a **hard limit** on metrics that affect user experience. If any metric exceeds the budget, it triggers a warning or blocks the deployment.

```
Types of Performance Budgets:
┌───────────────────────────────────────────────────────────┐
│                                                           │
│  1. TIMING BUDGETS (user-perceived speed)                 │
│     • Time to First Byte (TTFB): < 200ms                │
│     • First Contentful Paint (FCP): < 1.8s               │
│     • Largest Contentful Paint (LCP): < 2.5s             │
│     • Time to Interactive (TTI): < 3.5s                  │
│                                                           │
│  2. SIZE BUDGETS (what gets downloaded)                   │
│     • Total page weight: < 1MB                           │
│     • JavaScript bundle: < 300KB                         │
│     • Images total: < 500KB                              │
│     • CSS: < 100KB                                       │
│                                                           │
│  3. COUNT BUDGETS (number of resources)                   │
│     • HTTP requests: < 50                                │
│     • Third-party scripts: < 5                           │
│     • Web fonts: < 3                                     │
│                                                           │
│  4. RULE-BASED BUDGETS (Lighthouse scores)               │
│     • Performance score: > 90                            │
│     • Accessibility score: > 95                          │
│     • Best Practices: > 90                               │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Step 2: Core Web Vitals — Google's Official Metrics

Google uses three metrics to measure real user experience, and they **directly affect search rankings**:

```
┌──────────────────────────────────────────────────────────────┐
│              CORE WEB VITALS (2024)                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  LCP — Largest Contentful Paint                              │
│  "How fast does the main content appear?"                    │
│  ┌─────────┬──────────────┬─────────────┐                  │
│  │  Good   │ Needs Work   │    Poor     │                  │
│  │ < 2.5s  │ 2.5s - 4.0s │   > 4.0s   │                  │
│  │  🟢     │     🟡       │    🔴      │                  │
│  └─────────┴──────────────┴─────────────┘                  │
│                                                              │
│  INP — Interaction to Next Paint                             │
│  "How fast does the page respond to user actions?"           │
│  ┌─────────┬──────────────┬─────────────┐                  │
│  │  Good   │ Needs Work   │    Poor     │                  │
│  │ < 200ms │ 200ms-500ms  │  > 500ms   │                  │
│  │  🟢     │     🟡       │    🔴      │                  │
│  └─────────┴──────────────┴─────────────┘                  │
│                                                              │
│  CLS — Cumulative Layout Shift                              │
│  "Does content jump around while loading?"                   │
│  ┌─────────┬──────────────┬─────────────┐                  │
│  │  Good   │ Needs Work   │    Poor     │                  │
│  │ < 0.1   │  0.1 - 0.25  │   > 0.25   │                  │
│  │  🟢     │     🟡       │    🔴      │                  │
│  └─────────┴──────────────┴─────────────┘                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Step 3: Understanding Each Core Web Vital

#### LCP — Largest Contentful Paint

```
Page loading timeline:
  0s        1s        2s        3s        4s
  │─────────│─────────│─────────│─────────│
  │                                        │
  │  ┌──────┐                              │
  │  │ Nav  │ ← First paint (small)       │
  │  ├──────┤                              │
  │  │      │                              │
  │  │      │  ┌──────────────────┐       │
  │  │      │  │                  │       │
  │  │      │  │  HERO IMAGE     │ ← LCP! (largest element)
  │  │      │  │  (or main text) │       │
  │  │      │  │                  │       │
  │  │      │  └──────────────────┘       │
  │  │      │                              │
  └──┴──────┴──────────────────────────────┘
      ▲                ▲
      FCP              LCP = 2.1s ✅ (under 2.5s)

What counts as LCP element:
  • <img> elements
  • <video> poster images
  • Elements with background-image (CSS)
  • Block-level text elements (paragraphs, headings)
```

#### INP — Interaction to Next Paint

```
User clicks a button:
  │
  │  Click event fires
  ▼
  ┌──────────────────────────────────────┐
  │  INPUT DELAY     │ PROCESSING │ PRESENTATION │
  │  (waiting for    │ (event     │  (browser    │
  │   main thread)   │  handler)  │   repaints)  │
  └──────────────────────────────────────┘
  │◀─────────── INP (total time) ──────────────▶│
  
  Example:
    Input delay:    50ms (main thread was busy with JS)
    Processing:     80ms (your click handler runs)
    Presentation:   20ms (browser updates DOM + repaints)
    ─────────────────
    INP:           150ms ✅ (under 200ms)

  Bad example:
    Input delay:   200ms (huge JS bundle blocking main thread!)
    Processing:    300ms (heavy computation in click handler)
    Presentation:   50ms
    ─────────────────
    INP:           550ms ❌ (over 500ms — terrible!)
```

#### CLS — Cumulative Layout Shift

```
BAD CLS (content jumps around):                
┌────────────────────┐    ┌────────────────────┐
│ ┌────────────────┐ │    │ ┌──AD BANNER───────┐│ ← Ad loads, pushes
│ │   Article      │ │    │ └──────────────────┘│   content DOWN!
│ │   Content      │ │    │ ┌────────────────┐ │
│ │   Here         │ │    │ │   Article      │ │
│ │                │ │    │ │   Content      │ │
│ │   [Button]     │ │    │ │   Here         │ │
│ └────────────────┘ │    │ │                │ │
│                    │    │ │   [Button]     │ │ ← User was about
└────────────────────┘    └────────────────────┘   to click! Misclick!
   Before ad loads           After ad loads
   
CLS = (shifted area × distance shifted) / viewport area
     = (0.6 × 0.3) = 0.18 ← Bad! (> 0.1)

GOOD CLS (space reserved):
┌────────────────────┐    ┌────────────────────┐
│ ┌──placeholder────┐│    │ ┌──AD BANNER───────┐│
│ │  (fixed height) ││    │ └──────────────────┘│ ← Fills reserved
│ └─────────────────┘│    │ ┌────────────────┐ │   space, nothing shifts!
│ ┌────────────────┐ │    │ │   Article      │ │
│ │   Article      │ │    │ │   Content      │ │
│ │   Content      │ │    │ │   Here         │ │
└────────────────────┘    └────────────────────┘
CLS = 0 ✅
```

### Step 4: Setting Your Performance Budget

```
Step-by-step budget creation:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. MEASURE CURRENT STATE                                   │
│     Run Lighthouse on your site. Record baseline numbers.   │
│                                                             │
│  2. RESEARCH COMPETITORS                                    │
│     Check competitor page speeds (WebPageTest)              │
│     Your goal: be 20% faster than competitors              │
│                                                             │
│  3. SET TARGETS BASED ON USER IMPACT                        │
│     • 53% of mobile users abandon sites > 3s load          │
│     • Each 100ms delay = 1% less conversion (Amazon)        │
│     • Google ranks faster sites higher                      │
│                                                             │
│  4. DEFINE BUDGETS                                          │
│     ┌─────────────────────────────────────────────┐        │
│     │ Metric              │ Budget   │ Critical?  │        │
│     ├─────────────────────┼──────────┼────────────┤        │
│     │ LCP                 │ < 2.5s   │ Yes        │        │
│     │ INP                 │ < 200ms  │ Yes        │        │
│     │ CLS                 │ < 0.1    │ Yes        │        │
│     │ Total JS (gzipped)  │ < 300KB  │ Yes        │        │
│     │ Total page weight   │ < 1.5MB  │ Warning    │        │
│     │ Third-party scripts │ < 5      │ Warning    │        │
│     │ Lighthouse perf     │ > 90     │ Yes        │        │
│     └─────────────────────┴──────────┴────────────┘        │
│                                                             │
│  5. ENFORCE IN CI/CD                                        │
│     Build fails if budget is exceeded!                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Step 5: Measuring in the Field (RUM) vs Lab

```
┌──────────────────────────┬──────────────────────────┐
│      LAB DATA            │      FIELD DATA (RUM)    │
│  (synthetic testing)     │  (Real User Monitoring)  │
├──────────────────────────┼──────────────────────────┤
│                          │                          │
│ Lighthouse, WebPageTest  │ Chrome UX Report (CrUX)  │
│                          │ Google Analytics          │
│ Controlled environment   │ Real users, real devices  │
│ Same device, same network│ Varied networks/devices  │
│                          │                          │
│ Reproducible             │ Statistical (percentiles)│
│ Good for debugging       │ Shows actual user impact │
│                          │                          │
│ Doesn't show real user   │ THE metric Google uses   │
│ experience variety       │ for rankings!            │
│                          │                          │
│ Use for: CI/CD gates,    │ Use for: business        │
│ development, debugging   │ decisions, actual UX     │
└──────────────────────────┴──────────────────────────┘

Best practice: Use BOTH
  • Lab data in CI/CD (catch regressions before deploy)
  • Field data in production (understand real user experience)
```

---

## How It Works Internally

### How Browsers Measure LCP

```
Browser rendering pipeline:
  
  HTML arrives → Parse DOM → Build Render Tree → Layout → Paint
  
  LCP algorithm:
  1. Browser starts a "largest contentful paint" observer
  2. Every time a new element is painted, check its size
  3. If it's larger than the current LCP candidate → update LCP
  4. Stop tracking when user first interacts (click/tap/keypress)
  
  ┌──────────────────────────────────────────────────┐
  │  Time    │ Painted Element    │ Size    │ LCP?   │
  ├──────────┼────────────────────┼─────────┼────────┤
  │  200ms   │ Navigation bar     │ 50x800  │ Yes*   │
  │  500ms   │ Title <h1>         │ 200x600 │ Yes*   │
  │  1200ms  │ Hero image         │ 400x800 │ Yes*   │
  │  1500ms  │ (user scrolls)     │  -      │ STOP   │
  └──────────┴────────────────────┴─────────┴────────┘
  * Each replaces previous. Final LCP = 1200ms (hero image)
```

### How INP is Calculated

```
INP tracks ALL interactions during the page lifecycle:

  Interaction 1 (click):     120ms
  Interaction 2 (keypress):   50ms
  Interaction 3 (click):     380ms  ← worst
  Interaction 4 (scroll):     30ms
  Interaction 5 (click):     150ms
  
  INP = p98 of all interactions (not the worst, but near-worst)
  
  For pages with < 50 interactions: INP = single worst interaction
  For pages with 50+ interactions: INP = 98th percentile
  
  In this example: INP ≈ 380ms (the near-worst interaction)
```

### How CLS is Calculated

```
Layout Shift Score = Impact Fraction × Distance Fraction

Impact Fraction:
  The area of the viewport that shifted
  (Union of old position + new position / viewport)

Distance Fraction:  
  How far the element moved / viewport height

Example:
  Element covers 60% of viewport
  Moves down by 25% of viewport height
  
  CLS contribution = 0.60 × 0.25 = 0.15

Session Windows (since CLS update 2024):
  Shifts within 5 seconds of each other are grouped
  Max window duration: 5s
  Gap between shifts to start new window: 1s
  CLS = maximum session window value (not cumulative forever!)
```

---

## Code Examples

### Python — Performance Budget CI Checker

```python
import json
import subprocess
import sys

# Performance budget configuration
BUDGETS = {
    "lighthouse_performance": {"min": 90, "critical": True},
    "largest_contentful_paint_ms": {"max": 2500, "critical": True},
    "interaction_to_next_paint_ms": {"max": 200, "critical": True},
    "cumulative_layout_shift": {"max": 0.1, "critical": True},
    "total_byte_weight": {"max": 1_500_000, "critical": False},  # 1.5MB
    "script_byte_weight": {"max": 300_000, "critical": True},    # 300KB
    "third_party_requests": {"max": 5, "critical": False},
}

def run_lighthouse(url: str) -> dict:
    """Run Lighthouse and return results"""
    result = subprocess.run(
        ["lighthouse", url, "--output=json", "--quiet",
         "--chrome-flags='--headless'"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)

def check_budgets(lighthouse_results: dict) -> list:
    """Check if results meet performance budgets"""
    violations = []
    audits = lighthouse_results.get("audits", {})
    categories = lighthouse_results.get("categories", {})
    
    # Check Lighthouse score
    perf_score = categories.get("performance", {}).get("score", 0) * 100
    budget = BUDGETS["lighthouse_performance"]
    if perf_score < budget["min"]:
        violations.append({
            "metric": "Lighthouse Performance Score",
            "value": perf_score,
            "budget": f">= {budget['min']}",
            "critical": budget["critical"]
        })
    
    # Check LCP
    lcp = audits.get("largest-contentful-paint", {}).get("numericValue", 0)
    budget = BUDGETS["largest_contentful_paint_ms"]
    if lcp > budget["max"]:
        violations.append({
            "metric": "LCP",
            "value": f"{lcp:.0f}ms",
            "budget": f"<= {budget['max']}ms",
            "critical": budget["critical"]
        })
    
    # Check total weight
    total_weight = audits.get("total-byte-weight", {}).get("numericValue", 0)
    budget = BUDGETS["total_byte_weight"]
    if total_weight > budget["max"]:
        violations.append({
            "metric": "Total Page Weight",
            "value": f"{total_weight/1024:.0f}KB",
            "budget": f"<= {budget['max']/1024:.0f}KB",
            "critical": budget["critical"]
        })
    
    return violations

def enforce_budget(url: str):
    """Run budget check — exit with error if critical violation"""
    print(f"🔍 Checking performance budget for: {url}")
    results = run_lighthouse(url)
    violations = check_budgets(results)
    
    if not violations:
        print("✅ All performance budgets met!")
        return
    
    critical_failures = [v for v in violations if v["critical"]]
    warnings = [v for v in violations if not v["critical"]]
    
    for w in warnings:
        print(f"⚠️  WARNING: {w['metric']} = {w['value']} (budget: {w['budget']})")
    
    for f in critical_failures:
        print(f"❌ FAILED: {f['metric']} = {f['value']} (budget: {f['budget']})")
    
    if critical_failures:
        print(f"\n💀 {len(critical_failures)} critical budget(s) exceeded. BUILD FAILED!")
        sys.exit(1)

# Usage: python check_budget.py https://staging.example.com
if __name__ == "__main__":
    enforce_budget(sys.argv[1])
```

### JavaScript — Web Vitals Measurement (Client-Side)

```javascript
// Install: npm install web-vitals
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

// Report to analytics
function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,       // 'LCP', 'INP', 'CLS'
    value: metric.value,     // Numeric value
    rating: metric.rating,   // 'good', 'needs-improvement', 'poor'
    delta: metric.delta,     // Change since last report
    id: metric.id,           // Unique per page load
    navigationType: metric.navigationType,
    url: window.location.href,
  });

  // Use sendBeacon for reliability (doesn't block unload)
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/analytics/vitals', body);
  } else {
    fetch('/api/analytics/vitals', { body, method: 'POST', keepalive: true });
  }
}

// Register Core Web Vitals observers
onLCP(sendToAnalytics);   // Largest Contentful Paint
onINP(sendToAnalytics);   // Interaction to Next Paint
onCLS(sendToAnalytics);   // Cumulative Layout Shift
onFCP(sendToAnalytics);   // First Contentful Paint (supplemental)
onTTFB(sendToAnalytics);  // Time to First Byte (supplemental)

// Custom performance budget check (client-side warning)
onLCP((metric) => {
  if (metric.value > 2500) {
    console.warn(`⚠️ LCP Budget Exceeded: ${metric.value}ms (budget: 2500ms)`);
    console.warn(`LCP Element:`, metric.entries[0]?.element);
  }
});

onINP((metric) => {
  if (metric.value > 200) {
    console.warn(`⚠️ INP Budget Exceeded: ${metric.value}ms (budget: 200ms)`);
    // Log which interaction was slow
    const entry = metric.entries[0];
    console.warn(`Slow interaction: ${entry?.name} on`, entry?.target);
  }
});
```

### Java — Server-Side Performance Budget (Spring Boot)

```java
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import jakarta.servlet.http.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class PerformanceBudgetInterceptor implements HandlerInterceptor {

    // Server-side timing budgets
    private static final long TTFB_BUDGET_MS = 200;
    private static final long API_RESPONSE_BUDGET_MS = 500;
    
    private final ConcurrentHashMap<String, AtomicLong> violations = new ConcurrentHashMap<>();

    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, Object handler) {
        request.setAttribute("startTime", System.nanoTime());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, Exception ex) {
        long start = (long) request.getAttribute("startTime");
        long durationMs = (System.nanoTime() - start) / 1_000_000;
        
        String endpoint = request.getMethod() + " " + request.getRequestURI();
        
        // Check against budget
        if (durationMs > API_RESPONSE_BUDGET_MS) {
            violations.computeIfAbsent(endpoint, k -> new AtomicLong())
                      .incrementAndGet();
            
            // Add Server-Timing header for debugging
            response.addHeader("Server-Timing", 
                String.format("total;dur=%d;desc=\"OVER BUDGET (max %dms)\"",
                    durationMs, API_RESPONSE_BUDGET_MS));
            
            // Log warning
            System.out.printf("⚠️ BUDGET EXCEEDED: %s took %dms (budget: %dms)%n",
                endpoint, durationMs, API_RESPONSE_BUDGET_MS);
        } else {
            response.addHeader("Server-Timing", 
                String.format("total;dur=%d", durationMs));
        }
    }
}
```

---

## Infrastructure Examples

### Lighthouse CI in GitHub Actions

```yaml
# .github/workflows/performance-budget.yml
name: Performance Budget Check

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: npm ci && npm run build
      
      - name: Start server
        run: npm run preview &
        
      - name: Wait for server
        run: npx wait-on http://localhost:3000

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v11
        with:
          urls: |
            http://localhost:3000/
            http://localhost:3000/products
            http://localhost:3000/checkout
          budgetPath: ./lighthouse-budget.json
          uploadArtifacts: true

      - name: Check budget results
        if: failure()
        run: echo "❌ Performance budget exceeded! Check Lighthouse report."
```

```json
// lighthouse-budget.json
[
  {
    "path": "/*",
    "timings": [
      { "metric": "largest-contentful-paint", "budget": 2500 },
      { "metric": "first-contentful-paint", "budget": 1800 },
      { "metric": "interactive", "budget": 3500 },
      { "metric": "cumulative-layout-shift", "budget": 0.1 },
      { "metric": "total-blocking-time", "budget": 200 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 300 },
      { "resourceType": "stylesheet", "budget": 100 },
      { "resourceType": "image", "budget": 500 },
      { "resourceType": "total", "budget": 1500 }
    ],
    "resourceCounts": [
      { "resourceType": "third-party", "budget": 5 },
      { "resourceType": "script", "budget": 10 },
      { "resourceType": "total", "budget": 50 }
    ]
  }
]
```

### Webpack Bundle Size Budget

```javascript
// webpack.config.js — Enforce bundle size budgets
module.exports = {
  performance: {
    maxAssetSize: 300000,      // 300KB per asset (warning)
    maxEntrypointSize: 500000, // 500KB total entry (warning)
    hints: 'error',            // 'warning' or 'error' (fails build!)
    assetFilter: (assetFilename) => {
      // Only check JS and CSS files
      return /\.(js|css)$/.test(assetFilename);
    },
  },
};
```

### Real User Monitoring (RUM) Dashboard

```yaml
# Grafana dashboard — tracking Core Web Vitals from real users

# Prometheus metrics exposed by your analytics endpoint
panels:
  - title: "LCP (p75) by Page"
    query: |
      histogram_quantile(0.75, 
        rate(web_vital_lcp_bucket{page=~".*"}[1h])
      )
    thresholds:
      - value: 2500
        color: green
      - value: 4000
        color: yellow  
      - value: 999999
        color: red

  - title: "INP (p75) by Page"
    query: |
      histogram_quantile(0.75,
        rate(web_vital_inp_bucket{page=~".*"}[1h])
      )
    thresholds:
      - value: 200
        color: green
      - value: 500
        color: yellow
      - value: 999999
        color: red

  - title: "CLS (p75) by Page"
    query: |
      histogram_quantile(0.75,
        rate(web_vital_cls_bucket{page=~".*"}[1h])
      )
    thresholds:
      - value: 0.1
        color: green
      - value: 0.25
        color: yellow
      - value: 999999
        color: red
```

---

## Real-World Example

### Google — How Core Web Vitals Affect Rankings

```
Google's page experience ranking signal:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Google uses FIELD data (CrUX) from real Chrome users:  │
│                                                         │
│  • 75th percentile of all page visits in last 28 days  │
│  • Must pass ALL THREE Core Web Vitals:                │
│    ✅ LCP < 2.5s                                       │
│    ✅ INP < 200ms                                      │
│    ✅ CLS < 0.1                                        │
│                                                         │
│  Impact on rankings:                                    │
│  • Not a dominant factor (content relevance wins)       │
│  • TIE-BREAKER between equally relevant pages           │
│  • Can be difference between position 3 and 8          │
│                                                         │
│  SERP visual indicator:                                 │
│  Sites passing all CWV get a "page experience" badge   │
│                                                         │
│  % of origins passing CWV (2024):                      │
│    Mobile: ~42%                                         │
│    Desktop: ~57%                                        │
│    → Passing puts you ahead of majority!               │
└─────────────────────────────────────────────────────────┘
```

### Pinterest — Performance Budget Impact on Business

```
Pinterest's performance budget story:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Problem: Page load times crept up over months          │
│  (death by a thousand cuts — each feature added ms)     │
│                                                         │
│  Solution: Implemented strict performance budgets       │
│                                                         │
│  Budget: Page load < 3 seconds (mobile 3G)             │
│                                                         │
│  After enforcing budgets:                               │
│  • 40% reduction in wait time                          │
│  • 15% increase in SEO traffic                         │
│  • 15% increase in conversion to sign-up              │
│  • Revenue increase: millions per year                 │
│                                                         │
│  Enforcement:                                           │
│  • Lighthouse CI blocks PRs that exceed budget         │
│  • Bundle size check in every build                    │
│  • Third-party script review board                     │
│  • Quarterly "performance cleanup" sprints             │
└─────────────────────────────────────────────────────────┘
```

### Netflix — Loading Performance

```
Netflix optimization priorities:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Homepage TTI budget: < 2 seconds (on fast connection) │
│                                                         │
│  How they achieve it:                                   │
│  • Prefetch next-likely content during browse          │
│  • Lazy-load below-fold images                         │
│  • Inline critical CSS (no render-blocking CSS)        │
│  • Server-side render above-fold content               │
│  • JavaScript budget: < 150KB for initial load         │
│  • Defer non-critical JS with requestIdleCallback      │
│                                                         │
│  Measurement:                                           │
│  • Real User Monitoring on 200M+ devices               │
│  • A/B test performance changes for business impact    │
│  • Custom speed index per device category              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Optimization Strategies for Each Vital

```
IMPROVING LCP:
┌─────────────────────────────────────────────────────────┐
│ 1. Optimize largest element (hero image/main text)      │
│    • Use responsive images (srcset)                    │
│    • Preload LCP image: <link rel="preload">          │
│    • Use modern formats (WebP, AVIF)                  │
│ 2. Reduce server response time (TTFB < 200ms)         │
│ 3. Eliminate render-blocking CSS/JS                     │
│ 4. Don't lazy-load the LCP element!                    │
└─────────────────────────────────────────────────────────┘

IMPROVING INP:
┌─────────────────────────────────────────────────────────┐
│ 1. Break up long tasks (> 50ms) into smaller chunks    │
│    • Use requestIdleCallback / scheduler.yield()       │
│ 2. Reduce JavaScript bundle size                        │
│ 3. Use web workers for heavy computation               │
│ 4. Debounce input handlers                             │
│ 5. Avoid forced reflows (read DOM → batch writes)      │
└─────────────────────────────────────────────────────────┘

IMPROVING CLS:
┌─────────────────────────────────────────────────────────┐
│ 1. Set explicit dimensions on images/videos            │
│    • <img width="800" height="600">                   │
│    • CSS aspect-ratio property                         │
│ 2. Reserve space for ads/embeds                        │
│ 3. Avoid inserting content above existing content       │
│ 4. Use CSS transform animations (not layout properties)│
│ 5. Preload fonts to prevent FOIT/FOUT shifts          │
└─────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Only measuring lab data | Lab ≠ real user experience (different devices/networks) | Use CrUX/RUM for actual user metrics |
| Budget too lenient | "< 10 seconds" isn't a useful budget in 2024 | Use Google's Core Web Vitals thresholds as minimum |
| Only checking homepage | Internal pages often worse than homepage | Budget ALL critical user journeys |
| No enforcement mechanism | Budget without enforcement is just a wish | Fail CI/CD builds on budget violations |
| Lazy-loading the LCP element | Delays the most important content! | Never lazy-load above-the-fold / LCP elements |
| Ignoring third-party scripts | Analytics, ads, chat widgets add 200-500ms+ | Audit and defer all third-party scripts |
| Setting budget once, never updating | Performance targets should get STRICTER over time | Review budgets quarterly as you optimize |

---

## When to Use / When NOT to Use

### Set performance budgets WHEN:
- You have a user-facing website (especially e-commerce, media, SaaS)
- You have multiple developers adding features (prevents creep)
- SEO matters (Core Web Vitals affect Google rankings)
- You're on mobile-first sites (users have limited bandwidth)
- You need accountability for performance

### Less critical WHEN:
- Internal tools with few users on fast networks
- CLI tools or backend services (focus on latency/throughput instead)
- Prototype/MVP stage (get working first, optimize later)
- Single developer project (you are the enforcement)

---

## Key Takeaways

- **Performance budgets** are hard limits on speed metrics that block deploys when exceeded — like financial budgets for loading speed
- **Core Web Vitals** (LCP, INP, CLS) are Google's official UX metrics that **directly affect search rankings**
- **LCP < 2.5s** (main content visible), **INP < 200ms** (responsive to interaction), **CLS < 0.1** (no layout jumps)
- Measure with **both lab data** (Lighthouse CI) and **field data** (RUM/CrUX) — they answer different questions
- **Enforce budgets in CI/CD** — a budget without enforcement will be violated within weeks
- Companies like Pinterest saw **15% traffic increase** just from meeting performance budgets
- Performance degrades slowly ("death by a thousand cuts") — budgets prevent gradual creep

---

## What's Next?

Congratulations! You've completed **Part 19: Performance Engineering**. You now have a comprehensive toolkit:
- Measuring performance (latency, throughput, percentiles)
- Finding bottlenecks (profiling, flame graphs)
- Testing at scale (load testing, stress testing)
- Database optimization (indexes, N+1, query tuning)
- Transfer optimization (compression, minification)
- Protocol optimization (HTTP/2, HTTP/3)
- Tracking and enforcing performance (budgets, Core Web Vitals)

Next up is **Part 20: Data Engineering in Web Apps**, starting with **Chapter 20.1: OLTP vs OLAP — Transactional vs Analytical** — where you'll learn how production systems handle both real-time transactions AND massive analytical queries without one slowing down the other.
