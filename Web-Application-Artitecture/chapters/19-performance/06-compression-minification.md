# Compression (Gzip, Brotli) & Minification

> **What you'll learn**: How to shrink your HTTP responses by 60-90% using compression algorithms and code minification — making your app dramatically faster without changing any business logic.

---

## Real-Life Analogy

Imagine you need to **mail a large box of clothes** to a friend:

- **Without compression**: You stuff clothes loosely in a giant box. It's expensive to ship (heavy + bulky).
- **With compression (Gzip)**: You use a vacuum bag to suck out all the air. Same clothes, half the box size!
- **With better compression (Brotli)**: You use an even better vacuum bag with tighter seals. 30% smaller than Gzip!
- **With minification**: Before packing, you remove all the hangers, tissue paper, and tags that your friend doesn't need. Less stuff to pack in the first place!

```
Original Response: 250 KB
    │
    ├── Minification (remove whitespace/comments): 180 KB (-28%)
    │
    ├── Gzip compression: 45 KB (-82% from original!)
    │
    └── Brotli compression: 38 KB (-85% from original!)

Network transfer comparison (on 3G: ~1 Mbps):
  Original:    250KB ÷ 1Mbps = 2.0 seconds
  Gzip:         45KB ÷ 1Mbps = 0.36 seconds  ← 5.5x faster!
  Brotli:       38KB ÷ 1Mbps = 0.30 seconds  ← 6.6x faster!
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is HTTP Compression?

When a browser requests a page, it tells the server: "Hey, I understand compressed formats!" The server then compresses the response before sending it over the network. The browser decompresses it on arrival.

```
┌─────────┐                                    ┌──────────┐
│ Browser │─── Request ────────────────────────▶│  Server  │
│         │  Accept-Encoding: gzip, br          │          │
│         │                                     │          │
│         │                                     │ Compress │
│         │                                     │ response │
│         │                                     │          │
│         │◀── Response ───────────────────────│          │
│         │  Content-Encoding: br               │          │
│         │  (38KB instead of 250KB!)           │          │
│         │                                     │          │
│Decompress│                                    │          │
│& render │                                     │          │
└─────────┘                                    └──────────┘
```

### Step 2: How Content Negotiation Works

```
1. Browser sends request with supported encodings:
   GET /api/products HTTP/1.1
   Host: example.com
   Accept-Encoding: gzip, deflate, br    ← "I support these!"

2. Server picks the BEST encoding it supports:
   HTTP/1.1 200 OK
   Content-Encoding: br                   ← "I'm sending Brotli"
   Content-Type: application/json
   Content-Length: 38421                   ← Compressed size
   
   [compressed bytes...]

3. Browser decompresses using the specified algorithm

Priority order (best to worst):
  br (Brotli) > gzip > deflate > identity (no compression)
```

### Step 3: Gzip vs Brotli vs Zstandard

```
┌───────────────┬────────────────┬────────────────┬────────────────┐
│   Feature     │     Gzip       │    Brotli      │   Zstandard    │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ Compression   │ Good           │ Better (5-20%  │ Similar to     │
│ ratio         │                │ smaller)       │ Brotli         │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ Compress speed│ Fast           │ Slow at high   │ Very fast      │
│               │                │ levels         │                │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ Decompress    │ Fast           │ Fast           │ Very fast      │
│ speed         │                │                │                │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ Browser       │ All browsers   │ All modern     │ Limited        │
│ support       │ (since forever)│ (95%+)         │ (growing)      │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ Best for      │ Dynamic content│ Static assets  │ Real-time      │
│               │ (APIs)         │ (pre-compress) │ streaming      │
├───────────────┼────────────────┼────────────────┼────────────────┤
│ HTTP header   │ gzip           │ br             │ zstd           │
└───────────────┴────────────────┴────────────────┴────────────────┘
```

### Step 4: What is Minification?

**Minification** removes unnecessary characters from code WITHOUT changing functionality:

```javascript
// BEFORE minification (readable, 847 bytes):
function calculateShippingCost(items, destination) {
    // Calculate total weight of all items
    let totalWeight = 0;
    for (let i = 0; i < items.length; i++) {
        totalWeight += items[i].weight;
    }
    
    // Apply distance multiplier
    const distanceMultiplier = getDistanceMultiplier(destination);
    
    // Base rate is $5 per kg
    const baseCost = totalWeight * 5;
    
    return baseCost * distanceMultiplier;
}

// AFTER minification (same functionality, 142 bytes — 83% smaller!):
function calculateShippingCost(t,n){let e=0;for(let r=0;r<t.length;r++)e+=t[r].weight;return e*5*getDistanceMultiplier(n)}
```

What minification removes:
- Comments (`// ...`, `/* ... */`)
- Whitespace and newlines
- Long variable names → short ones (`totalWeight` → `e`)
- Unnecessary semicolons and brackets

### Step 5: Types of Minification

```
┌──────────────────────────────────────────────────────────────┐
│                MINIFICATION TARGETS                           │
├──────────────┬───────────────────────────────────────────────┤
│ JavaScript   │ Remove comments, whitespace, shorten vars     │
│              │ Tools: Terser, UglifyJS, esbuild              │
│              │ Savings: 40-70%                               │
├──────────────┼───────────────────────────────────────────────┤
│ CSS          │ Remove comments, whitespace, merge rules      │
│              │ Tools: cssnano, clean-css, Lightning CSS      │
│              │ Savings: 30-60%                               │
├──────────────┼───────────────────────────────────────────────┤
│ HTML         │ Remove comments, optional tags, whitespace    │
│              │ Tools: html-minifier-terser                   │
│              │ Savings: 15-30%                               │
├──────────────┼───────────────────────────────────────────────┤
│ JSON         │ Remove whitespace (pretty-print → compact)    │
│              │ No tools needed: json.dumps(data)             │
│              │ Savings: 20-40%                               │
├──────────────┼───────────────────────────────────────────────┤
│ Images       │ Not "minification" but optimization           │
│              │ Tools: Sharp, ImageMagick, WebP/AVIF format   │
│              │ Savings: 30-80%                               │
└──────────────┴───────────────────────────────────────────────┘
```

### Step 6: The Complete Optimization Pipeline

```
Source Code
    │
    ▼
┌──────────────────┐
│  1. Minification │  Remove comments, whitespace, shorten names
│     (build time) │  847 bytes → 142 bytes
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. Bundling     │  Combine 50 files into 2-3 bundles
│     (build time) │  Reduces HTTP requests
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. Pre-compress │  Generate .gz and .br versions at build time
│     (build time) │  142 bytes → 98 bytes (br)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. Serve        │  Nginx/CDN serves pre-compressed file
│     (runtime)    │  No CPU cost per request!
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. Cache        │  Browser caches compressed response
│     (client)     │  Next visit: 0 bytes transferred!
└──────────────────┘
```

---

## How It Works Internally

### How Gzip (DEFLATE) Works

Gzip uses two techniques:
1. **LZ77** — Find repeated patterns and replace with back-references
2. **Huffman coding** — Use fewer bits for common characters

```
Original text:
  "the cat sat on the mat the cat sat"

Step 1: LZ77 — Find repeated sequences
  "the cat sat on [the](ref:0,3) mat [the cat sat](ref:0,11)"
  
  Repeated "the" → reference to earlier occurrence
  Repeated "the cat sat" → reference to earlier occurrence

Step 2: Huffman — Common characters get short codes
  'a' (frequent) → 2 bits: 10
  't' (frequent) → 3 bits: 110
  'z' (rare)     → 8 bits: 11010110
  
  More common characters = fewer bits = smaller output!

Result: 35 bytes → ~20 bytes (43% smaller)
At larger scales (KB of HTML): 60-80% reduction typical
```

### How Brotli Improves on Gzip

```
Brotli's advantages:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Bigger sliding window (16MB vs Gzip's 32KB)        │
│     → Can reference patterns much further back          │
│                                                         │
│  2. Built-in dictionary of common web strings:          │
│     "Content-Type", "application/json", "<!DOCTYPE",    │
│     "function", "return", ".com", "http://", etc.       │
│     → Doesn't need to "learn" common web patterns!     │
│                                                         │
│  3. Context modeling                                    │
│     → Better prediction of next bytes                  │
│                                                         │
│  Result: 5-20% smaller than Gzip for web content       │
│  Tradeoff: Slower compression (2-5x slower at max)     │
│  Solution: Pre-compress static assets at build time!    │
└─────────────────────────────────────────────────────────┘
```

### What Gets Compressed Well vs Poorly

```
HIGH compression (text-based, repetitive):
  HTML:        80-85% reduction
  CSS:         75-85% reduction  
  JavaScript:  70-80% reduction
  JSON/XML:    75-85% reduction
  SVG:         70-80% reduction

LOW compression (already compressed/binary):
  JPEG/PNG:    0-5% reduction (already compressed!)
  MP4/WebM:    0-2% reduction
  WOFF2 fonts: 0-3% reduction (already Brotli-compressed!)
  ZIP files:   0% (already compressed)

Rule: Don't compress already-compressed formats!
  → Wastes CPU for zero benefit
  → Can actually make files LARGER!
```

---

## Code Examples

### Python — Flask with Compression

```python
from flask import Flask, jsonify, request, make_response
import gzip
import brotli
import json

app = Flask(__name__)

# Method 1: Using flask-compress (simplest)
# pip install flask-compress
from flask_compress import Compress
Compress(app)  # Automatically compresses all responses!

# Method 2: Manual compression (for understanding)
@app.route('/api/products')
def get_products():
    data = generate_large_product_list()  # Returns dict
    json_bytes = json.dumps(data).encode('utf-8')
    
    # Check what the client accepts
    accept_encoding = request.headers.get('Accept-Encoding', '')
    
    if 'br' in accept_encoding:
        # Brotli compression (best ratio)
        compressed = brotli.compress(json_bytes, quality=4)
        response = make_response(compressed)
        response.headers['Content-Encoding'] = 'br'
    elif 'gzip' in accept_encoding:
        # Gzip compression (universal support)
        compressed = gzip.compress(json_bytes, compresslevel=6)
        response = make_response(compressed)
        response.headers['Content-Encoding'] = 'gzip'
    else:
        response = make_response(json_bytes)
    
    response.headers['Content-Type'] = 'application/json'
    response.headers['Vary'] = 'Accept-Encoding'  # Important for CDN!
    
    original_size = len(json_bytes)
    compressed_size = len(response.get_data())
    print(f"Compression: {original_size}B → {compressed_size}B "
          f"({(1 - compressed_size/original_size)*100:.0f}% saved)")
    
    return response

# Method 3: Pre-compressed static files
import os
from pathlib import Path

def pre_compress_static_files(static_dir):
    """Pre-compress all JS/CSS/HTML files at build time"""
    for filepath in Path(static_dir).rglob('*'):
        if filepath.suffix in ('.js', '.css', '.html', '.json', '.svg'):
            content = filepath.read_bytes()
            
            # Generate .gz version
            gz_path = filepath.with_suffix(filepath.suffix + '.gz')
            gz_path.write_bytes(gzip.compress(content, compresslevel=9))
            
            # Generate .br version
            br_path = filepath.with_suffix(filepath.suffix + '.br')
            br_path.write_bytes(brotli.compress(content, quality=11))
            
            print(f"{filepath.name}: {len(content)}B → "
                  f"gz:{os.path.getsize(gz_path)}B, "
                  f"br:{os.path.getsize(br_path)}B")
```

### Java — Spring Boot with Compression

```java
import org.springframework.boot.web.server.Compression;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.*;
import java.util.zip.GZIPOutputStream;

@Configuration
public class CompressionConfig {

    // Method 1: Spring Boot built-in compression (simplest)
    // In application.yml:
    // server:
    //   compression:
    //     enabled: true
    //     min-response-size: 1024
    //     mime-types: application/json,text/html,text/css,application/javascript

    // Method 2: Custom compression filter (for fine control)
    @Bean
    public FilterRegistrationBean<CompressionFilter> compressionFilter() {
        FilterRegistrationBean<CompressionFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new CompressionFilter());
        bean.addUrlPatterns("/api/*");
        bean.setOrder(1);
        return bean;
    }
}

public class CompressionFilter implements Filter {
    
    private static final int MIN_SIZE = 1024; // Don't compress < 1KB

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;
        
        String acceptEncoding = request.getHeader("Accept-Encoding");
        
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            GzipResponseWrapper wrapper = new GzipResponseWrapper(response);
            chain.doFilter(request, wrapper);
            wrapper.finish();
            
            byte[] compressed = wrapper.getCompressedData();
            if (compressed.length < wrapper.getOriginalSize()) {
                response.setHeader("Content-Encoding", "gzip");
                response.setHeader("Vary", "Accept-Encoding");
                response.setContentLength(compressed.length);
                response.getOutputStream().write(compressed);
            }
        } else {
            chain.doFilter(request, response);
        }
    }
}

// Spring Boot application.yml configuration
/*
server:
  compression:
    enabled: true
    min-response-size: 1024
    mime-types:
      - application/json
      - application/xml
      - text/html
      - text/css
      - text/javascript
      - application/javascript
*/
```

---

## Infrastructure Examples

### Nginx — Compression Configuration

```nginx
# nginx.conf — Production compression setup

http {
    # Gzip (for dynamic content — compressed on-the-fly)
    gzip on;
    gzip_vary on;                    # Add Vary: Accept-Encoding header
    gzip_proxied any;                # Compress proxied responses too
    gzip_comp_level 6;               # Balance: speed vs ratio (1-9)
    gzip_min_length 1024;            # Don't compress tiny responses
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml
        application/wasm;

    # Brotli (requires ngx_brotli module)
    brotli on;
    brotli_comp_level 6;             # 1-11 (11=best ratio, slowest)
    brotli_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml;

    # Pre-compressed static files (BEST approach for static assets)
    # Serve .br or .gz files directly — zero CPU cost!
    server {
        location /static/ {
            # Try serving pre-compressed version first
            gzip_static on;          # Serves .gz files if they exist
            brotli_static on;        # Serves .br files if they exist
            
            expires 1y;              # Cache for 1 year (fingerprinted files)
            add_header Cache-Control "public, immutable";
        }
        
        location /api/ {
            # Dynamic responses — compress on-the-fly
            proxy_pass http://backend;
        }
    }
}
```

### CDN Configuration (CloudFront / Cloudflare)

```yaml
# AWS CloudFront — compression settings via Terraform
resource "aws_cloudfront_distribution" "cdn" {
  default_cache_behavior {
    # Enable automatic compression
    compress = true  # CloudFront compresses if origin doesn't
    
    viewer_protocol_policy = "redirect-to-https"
    
    # Cache based on Accept-Encoding header
    cache_policy_id = aws_cloudfront_cache_policy.optimized.id
  }
}

resource "aws_cloudfront_cache_policy" "optimized" {
  name = "optimized-compression"
  
  parameters_in_cache_key_and_forwarded_to_origin {
    headers_config {
      header_behavior = "whitelist"
      headers {
        items = ["Accept-Encoding"]  # Different cache per encoding
      }
    }
    
    enable_accept_encoding_gzip  = true
    enable_accept_encoding_brotli = true
  }
}
```

### Build Pipeline — Minification + Pre-compression

```json
// package.json — Build pipeline with compression
{
  "scripts": {
    "build": "npm run build:js && npm run build:css && npm run compress",
    "build:js": "esbuild src/app.js --bundle --minify --outfile=dist/app.min.js",
    "build:css": "lightningcss --minify --bundle src/styles.css -o dist/styles.min.css",
    "compress": "node scripts/precompress.js"
  }
}
```

```javascript
// scripts/precompress.js — Pre-compress all static assets
const fs = require('fs');
const path = require('path');
const zlib = require('zlib');
const { compress } = require('brotli');

const DIST_DIR = './dist';
const EXTENSIONS = ['.js', '.css', '.html', '.json', '.svg'];

function processDir(dir) {
    const files = fs.readdirSync(dir, { recursive: true });
    
    for (const file of files) {
        const filePath = path.join(dir, file);
        const ext = path.extname(file);
        
        if (!EXTENSIONS.includes(ext)) continue;
        
        const content = fs.readFileSync(filePath);
        
        // Gzip (level 9 = max compression, fine for build time)
        const gzipped = zlib.gzipSync(content, { level: 9 });
        fs.writeFileSync(`${filePath}.gz`, gzipped);
        
        // Brotli (quality 11 = max, slow but pre-computed)
        const brotlied = compress(content, { quality: 11 });
        fs.writeFileSync(`${filePath}.br`, Buffer.from(brotlied));
        
        const savings = ((1 - brotlied.length / content.length) * 100).toFixed(0);
        console.log(`${file}: ${content.length}B → br:${brotlied.length}B (${savings}% saved)`);
    }
}

processDir(DIST_DIR);
```

---

## Real-World Example

### Google — Compression at Hyperscale

```
Google serves ~8.5 BILLION searches per day.
Even 1KB saved per response = 8.5 PETABYTES/day saved in bandwidth!

Google's approach:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Brotli everywhere (Google invented Brotli!)         │
│     - All Google properties use Brotli level 11         │
│     - Saves 15-25% vs Gzip across google.com           │
│                                                         │
│  2. Pre-compressed at build time                        │
│     - HTML templates compressed at deploy               │
│     - JS/CSS fingerprinted + pre-compressed            │
│                                                         │
│  3. Shared dictionary compression (experimental)        │
│     - Same base JS framework → send only DIFF          │
│     - Can reduce 200KB → 5KB for repeat visits!        │
│                                                         │
│  4. Protocol-level compression (HTTP/2 HPACK)           │
│     - HTTP headers compressed too                       │
│     - Saves 30KB+ of headers on pages with many assets │
│                                                         │
│  Impact: Saves BILLIONS of dollars in bandwidth/year   │
└─────────────────────────────────────────────────────────┘
```

### LinkedIn — Minification + Compression Savings

```
LinkedIn's frontend optimization:
  - 2000+ JavaScript modules → bundled into 5 chunks
  - Minification: 12MB total → 4.2MB (-65%)
  - Brotli compression: 4.2MB → 980KB (-77% of minified!)
  - Total: 12MB → 980KB over the wire = 92% reduction!
  
  Impact on page load:
    Before: 8.5 seconds (3G network)
    After:  2.1 seconds (3G network) — 4x faster!
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Compressing images/videos | Already compressed formats — wastes CPU, no benefit | Only compress text-based formats (HTML, CSS, JS, JSON, SVG) |
| Compressing tiny responses (<1KB) | Compression overhead (headers + CPU) exceeds savings | Set min-response-size to 1024 bytes |
| Using Brotli level 11 for dynamic responses | Too slow (100ms+ to compress) — adds latency | Use level 4-6 for dynamic; level 11 only for pre-compressed static |
| Not setting `Vary: Accept-Encoding` | CDN serves compressed version to client that doesn't support it | Always include Vary header |
| Compressing on app server | Wastes app server CPU that should serve requests | Compress at reverse proxy (Nginx) or CDN level |
| Double-compressing | Content compressed twice becomes LARGER | Check if upstream already compressed |
| Not minifying before compressing | Compression is more effective on minified code | Always minify first, then compress |

---

## When to Use / When NOT to Use

### Enable compression WHEN:
- Serving text-based content (HTML, CSS, JS, JSON, XML, SVG)
- Response size is > 1KB
- You have a reverse proxy or CDN that can handle it
- Bandwidth is constrained (mobile users, global users)

### Skip compression WHEN:
- Response is already compressed (images, videos, WOFF2 fonts)
- Response is < 1KB (overhead not worth it)
- You're CPU-constrained with no reverse proxy (rare)
- Serving to clients that don't support encoding (ancient clients)

### Use pre-compression WHEN:
- Serving static assets that don't change between requests
- You want maximum compression ratio with zero runtime CPU cost
- Building a production deployment pipeline

---

## Key Takeaways

- **Gzip** reduces text-based responses by 60-80%; **Brotli** adds 5-20% more savings on top
- **Minification** (removing whitespace/comments) should happen BEFORE compression — they're complementary
- **Pre-compress static assets** at build time (Brotli level 11) for maximum savings with zero runtime CPU cost
- **Never compress** already-compressed formats (JPEG, PNG, MP4, ZIP)
- Compress at the **reverse proxy/CDN level** (Nginx, CloudFront), not in your application code
- Always set `Vary: Accept-Encoding` so CDNs cache different versions correctly
- At Google's scale, even 1KB saved per response = petabytes of bandwidth saved per day

---

## What's Next?

Compression reduces the SIZE of data sent over the wire, but what about the PROTOCOL used to send it? In **Chapter 19.7: HTTP/2 & HTTP/3 (QUIC) — Faster Protocols**, you'll learn how modern protocols eliminate head-of-line blocking, enable multiplexing, and make web communication fundamentally faster.
