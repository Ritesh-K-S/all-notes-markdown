# Monolithic with Separate UI & API (Decoupled Monolith)

> **What you'll learn**: How to split a monolith into a separate frontend and backend API while keeping the backend as a single deployable unit — the most common first step in architecture evolution.

---

## Real-Life Analogy

Remember our restaurant analogy from Chapter 4.1? Imagine the restaurant owner makes one smart decision: **separate the dining hall from the kitchen**.

Now the **dining hall** (frontend) can be redesigned, painted, or even moved to a new location — without touching the kitchen. The **kitchen** (backend API) keeps cooking food the same way, serving dishes through a **service window** (the API).

The kitchen is still one big kitchen (monolith), but now it communicates with the dining hall through a well-defined interface: "Give me dish #42" rather than the waiter walking into the kitchen and stirring the pot themselves.

---

## Core Concept Explained Step-by-Step

### What is a Decoupled Monolith?

A **decoupled monolith** separates the application into two main parts:
1. **Frontend Application** — HTML/CSS/JS (React, Angular, Vue) served independently
2. **Backend API** — A monolithic server exposing REST/GraphQL endpoints

They communicate over **HTTP/HTTPS** via well-defined API contracts.

### Before vs After

```
TRADITIONAL MONOLITH (Coupled):
┌─────────────────────────────────────────┐
│            SINGLE SERVER                │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │   Server-Side Templates (HTML)   │   │
│  │   (JSP, Thymeleaf, Jinja2, EJS) │   │
│  └──────────────┬───────────────────┘   │
│                 │ (direct call)          │
│  ┌──────────────▼───────────────────┐   │
│  │     Business Logic & API         │   │
│  └──────────────┬───────────────────┘   │
│                 │                        │
│  ┌──────────────▼───────────────────┐   │
│  │         Database Layer           │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘


DECOUPLED MONOLITH (Separate UI & API):
┌──────────────────┐         ┌─────────────────────────────┐
│   FRONTEND APP   │         │      BACKEND API            │
│                  │  HTTP    │       (Monolith)            │
│  React / Vue /   │────────▶│                             │
│  Angular / Next  │◀────────│  ┌───────────────────────┐  │
│                  │  JSON    │  │   Business Logic      │  │
│  Served from     │         │  └───────────┬───────────┘  │
│  CDN or Nginx    │         │              │              │
└──────────────────┘         │  ┌───────────▼───────────┐  │
                             │  │    Database Layer     │  │
                             │  └───────────────────────┘  │
                             └─────────────────────────────┘
```

### Why This Matters

This is often the **first architectural evolution** a growing application makes because it gives you:

| Benefit | Explanation |
|---|---|
| **Independent deployment** | Ship frontend changes without touching backend |
| **Team separation** | Frontend team ≠ Backend team, different skills |
| **Technology freedom** | Frontend can use React today, switch to Vue tomorrow |
| **Better performance** | Frontend served from CDN (global edge) |
| **Mobile-ready** | The same API serves web, iOS, and Android |

---

## How It Works Internally

### Communication Flow

```
┌─────────┐     ┌─────────────┐     ┌──────────────────────────┐
│ Browser │     │   CDN       │     │    Backend API Server     │
│         │     │ (Frontend)  │     │       (Monolith)          │
└────┬────┘     └──────┬──────┘     └────────────┬─────────────┘
     │                 │                          │
     │  1. GET /       │                          │
     │────────────────▶│                          │
     │  2. HTML+JS+CSS │                          │
     │◀────────────────│                          │
     │                 │                          │
     │  3. API Call: GET /api/products             │
     │────────────────────────────────────────────▶│
     │  4. JSON Response                           │
     │◀────────────────────────────────────────────│
     │                 │                          │
     │  5. POST /api/orders (with JSON body)       │
     │────────────────────────────────────────────▶│
     │  6. { "order_id": 123, "status": "ok" }    │
     │◀────────────────────────────────────────────│
     │                 │                          │
```

### The API Contract

The **API contract** is the glue between frontend and backend. It defines:
- Which endpoints exist (`GET /api/products`, `POST /api/orders`)
- What data format is expected (JSON request/response shapes)
- Authentication method (JWT tokens, cookies)
- Error formats

```
API CONTRACT EXAMPLE:

Endpoint: POST /api/orders
Headers:  Authorization: Bearer <JWT_TOKEN>
          Content-Type: application/json

Request Body:
{
  "product_id": 42,
  "quantity": 2,
  "shipping_address": "123 Main St"
}

Response (201 Created):
{
  "order_id": 1001,
  "status": "pending",
  "estimated_delivery": "2024-01-15"
}

Response (400 Bad Request):
{
  "error": "INSUFFICIENT_STOCK",
  "message": "Only 1 item available"
}
```

### CORS — The Bridge Between Domains

When frontend and backend are on different domains/ports, browsers enforce **CORS** (Cross-Origin Resource Sharing):

```
Frontend: https://app.mystore.com  (served from CDN)
Backend:  https://api.mystore.com  (API server)

Browser Request:
OPTIONS /api/orders HTTP/1.1
Origin: https://app.mystore.com

Server Response (CORS headers):
Access-Control-Allow-Origin: https://app.mystore.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
```

---

## Code Examples

### Python Backend (FastAPI)

```python
# backend/main.py - Monolithic API serving JSON
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from sqlalchemy.orm import Session

app = FastAPI()

# CORS: Allow frontend origin to call this API
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.mystore.com", "http://localhost:3000"],
    allow_methods=["*"],
    allow_headers=["*"],
)

class OrderRequest(BaseModel):
    product_id: int
    quantity: int
    shipping_address: str

@app.post("/api/orders", status_code=201)
def create_order(order: OrderRequest, db: Session = Depends(get_db)):
    # All business logic still in one monolithic backend
    product = db.query(Product).get(order.product_id)
    if not product or product.stock < order.quantity:
        raise HTTPException(400, "Insufficient stock")
    
    product.stock -= order.quantity
    new_order = Order(**order.dict(), status="pending")
    db.add(new_order)
    db.commit()
    
    return {"order_id": new_order.id, "status": "pending"}

@app.get("/api/products")
def list_products(db: Session = Depends(get_db)):
    return db.query(Product).filter(Product.active == True).all()
```

### React Frontend

```javascript
// frontend/src/components/OrderForm.jsx
import { useState } from 'react';

const API_BASE = process.env.REACT_APP_API_URL; // https://api.mystore.com

export function OrderForm({ productId }) {
  const [status, setStatus] = useState(null);

  const placeOrder = async () => {
    const response = await fetch(`${API_BASE}/api/orders`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('token')}`,
      },
      body: JSON.stringify({
        product_id: productId,
        quantity: 1,
        shipping_address: '123 Main St',
      }),
    });

    if (response.ok) {
      const data = await response.json();
      setStatus(`Order #${data.order_id} placed!`);
    } else {
      const error = await response.json();
      setStatus(`Error: ${error.message}`);
    }
  };

  return (
    <div>
      <button onClick={placeOrder}>Place Order</button>
      {status && <p>{status}</p>}
    </div>
  );
}
```

### Java Backend (Spring Boot)

```java
// backend/src/main/java/com/store/OrderController.java
@RestController
@RequestMapping("/api")
@CrossOrigin(origins = {"https://app.mystore.com", "http://localhost:3000"})
public class OrderController {

    @Autowired private ProductRepository productRepo;
    @Autowired private OrderRepository orderRepo;

    @PostMapping("/orders")
    @Transactional
    public ResponseEntity<?> createOrder(@RequestBody @Valid OrderRequest request) {
        Product product = productRepo.findById(request.getProductId())
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        if (product.getStock() < request.getQuantity()) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "INSUFFICIENT_STOCK"));
        }

        product.setStock(product.getStock() - request.getQuantity());
        productRepo.save(product);

        Order order = new Order(request.getProductId(), 
                               request.getQuantity(), "pending");
        orderRepo.save(order);

        return ResponseEntity.status(201)
            .body(Map.of("order_id", order.getId(), "status", "pending"));
    }

    @GetMapping("/products")
    public List<Product> listProducts() {
        return productRepo.findByActiveTrue();
    }
}
```

---

## Infrastructure Example

### Deployment Architecture

```yaml
# docker-compose.yml - Separate containers for frontend and backend
version: '3.8'
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"  # Nginx serving static React build
    environment:
      - REACT_APP_API_URL=http://localhost:8080

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://db:5432/ecommerce
      - JWT_SECRET=your-secret-key
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Production Setup with CDN

```
┌──────────────┐        ┌────────────────────┐
│   Browser    │        │   CloudFront CDN   │
│              │───(1)──▶  (Static Assets)   │
│              │◀──(2)──│  React build files  │
│              │        └────────────────────┘
│              │
│              │        ┌────────────────────┐     ┌────────────┐
│              │───(3)──▶  AWS ALB           │────▶│  Backend   │
│              │◀──(4)──│  (Load Balancer)   │◀────│  Monolith  │
└──────────────┘        └────────────────────┘     │  (ECS/EC2) │
                                                   └──────┬─────┘
                                                          │
                                                   ┌──────▼─────┐
                                                   │  RDS       │
                                                   │ PostgreSQL │
                                                   └────────────┘
```

### Nginx Configuration for Frontend

```nginx
# frontend/nginx.conf - Serves React SPA and proxies API
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Serve React SPA (all routes return index.html)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to backend (alternative to CORS)
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Real-World Example

### How Most Modern Companies Structure This

**GitHub** (before and after):
- **Before**: Ruby on Rails monolith rendered HTML server-side
- **After**: React frontend + Rails API backend (progressive migration)

**Airbnb**:
- React frontend deployed to CDN
- Monolithic Ruby on Rails API (later broken into services)
- Same API serves web + mobile apps

**Typical Startup Evolution**:

```
PHASE 1 (0-6 months):     PHASE 2 (6-18 months):     PHASE 3 (18+ months):
┌─────────────┐           ┌───────────┐               ┌─────────────┐
│  Monolith   │           │  React    │               │  React SPA  │
│  (Rails +   │    ──▶    │  Frontend │      ──▶      │  (CDN)      │
│   HTML)     │           ├───────────┤               ├─────────────┤
│             │           │  Rails    │               │  Service A  │
└─────────────┘           │  API      │               │  Service B  │
                          └───────────┘               │  Service C  │
                                                      └─────────────┘
   Everything               Decoupled                  Microservices
   together                 Monolith                   (later)
```

---

## Common Mistakes / Pitfalls

### 1. No API Versioning
❌ **Mistake**: Changing API response format breaks the deployed frontend.
✅ **Fix**: Version your API (`/api/v1/products`) or use backward-compatible changes.

### 2. Tight Frontend-Backend Coupling via API Shape
❌ **Mistake**: Frontend expects exact DB column names, creating coupling to internal structure.
✅ **Fix**: Use DTOs (Data Transfer Objects) — transform internal models before sending as JSON.

### 3. Authentication Complexity
❌ **Mistake**: Using server-side sessions when frontend is on a different domain.
✅ **Fix**: Use **JWT tokens** — the frontend stores the token and sends it with every request.

```
Authentication Flow:
┌──────────┐         ┌──────────────┐
│ Frontend │──(1)───▶│ POST /login  │
│          │◀──(2)───│ { token }    │
│          │         └──────────────┘
│          │
│  Store token in localStorage/memory
│          │
│          │──(3)───▶ GET /api/orders
│          │         Authorization: Bearer <token>
│          │◀──(4)───│ { orders: [...] }
└──────────┘         └──────────────┘
```

### 4. Over-fetching / Under-fetching
❌ **Mistake**: API returns too much data (over-fetching) or requires 5 calls for one page (under-fetching).
✅ **Fix**: Design APIs for frontend needs. Consider BFF pattern (Chapter 8.7) or GraphQL (Chapter 3.3).

### 5. Missing Error Contract
❌ **Mistake**: Frontend doesn't know how to handle backend errors (inconsistent error formats).
✅ **Fix**: Standardize error responses: `{ "error": "CODE", "message": "Human-readable" }`.

---

## When to Use / When NOT to Use

### ✅ Use Decoupled Monolith When:

| Criteria | Why |
|---|---|
| **Separate frontend & backend teams** | Each team can work independently |
| **Need mobile app support** | Same API serves web + mobile |
| **Want frontend on CDN** | Better global performance for static assets |
| **Preparing for future microservices** | Clean API boundary is the first step |
| **Different deploy cadences** | Frontend deploys 5x/day, backend 1x/day |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Very simple CRUD app** | Server-rendered templates are simpler |
| **SEO-critical with no SSR** | CSR-only frontends hurt SEO (unless using Next.js/Nuxt) |
| **Solo developer** | The overhead of two deployments isn't worth it alone |
| **Internal tools** | Admin dashboards don't need CDN or mobile support |

---

## Key Takeaways

- 🔀 **Decoupling = separating frontend deployment from backend deployment** while keeping the backend as a single monolith.
- 📡 **They communicate via HTTP APIs** (REST or GraphQL) with a well-defined contract.
- 🌍 **Frontend on CDN = fast globally** — static files served from edge locations worldwide.
- 📱 **One API, many clients** — the same backend serves web, iOS, Android, and even third-party integrations.
- 🔐 **JWT tokens solve cross-domain auth** — no more server-side session cookies.
- 🚀 **This is the natural first evolution** from a traditional monolith — low risk, high reward.
- ⚙️ **The backend is still a monolith** — all the benefits (simple transactions, easy debugging) and limitations (scaling, deployment coupling) remain.

---

## What's Next?

Now that we've separated the frontend, let's look at how to structure the backend itself more cleanly. In **Chapter 4.3: Layered (N-Tier) Architecture**, we'll explore how to organize your monolith's internals into clear layers — presentation, business logic, and data access — to keep code maintainable as it grows.
