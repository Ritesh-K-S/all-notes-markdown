# Chapter 3.2: REST APIs — The Standard Way to Talk to a Server

> **Level**: ⭐ Beginner  
> **What you'll learn**: How frontend and backend communicate using REST APIs — the most widely used API design pattern on the web, covering HTTP methods, status codes, URL design, and building your own API.

---

## 🧠 Real-Life Analogy: A Restaurant Menu

Imagine a restaurant where the waiter gives you a **menu**. The menu tells you exactly:
- What dishes are available
- How to order them
- What you'll get back

A REST API is like that menu — it tells your frontend exactly what data is available, how to ask for it, and what the response will look like.

```
    RESTAURANT                          REST API
    ══════════                          ════════
    
    Menu                                API Documentation
    (lists all dishes)                  (lists all endpoints)
    
    "I'd like the pasta"               GET /api/products/42
    (ordering a dish)                   (requesting a resource)
    
    Waiter brings pasta                 Server returns JSON
    (response)                          { "id": 42, "name": "Laptop" }
    
    "Add extra cheese"                  PUT /api/products/42
    (modifying order)                   (updating a resource)
    
    "Cancel my order"                   DELETE /api/products/42
    (removing order)                    (deleting a resource)
    
    
    The menu has RULES:
    • You can't order "give me everything in the kitchen"
    • Each dish has a specific name
    • The kitchen either has it or says "sorry, unavailable"
    
    REST APIs have RULES too:
    • Each resource has a specific URL
    • You use specific HTTP methods (GET, POST, PUT, DELETE)
    • The server responds with status codes (200 OK, 404 Not Found)
```

---

## 📖 What is REST?

**REST** stands for **RE**presentational **S**tate **T**ransfer. It's a set of rules (conventions) for designing web APIs that are simple, consistent, and scalable.

```
    REST is NOT a protocol or technology.
    REST is a set of CONVENTIONS for designing APIs.
    
    ┌──────────────────────────────────────────────────────────┐
    │  REST = How you DESIGN your API                          │
    │  HTTP = The PROTOCOL you use to communicate              │
    │  JSON = The FORMAT of the data (usually)                 │
    │                                                          │
    │  REST API = HTTP + JSON + URL conventions + status codes │
    └──────────────────────────────────────────────────────────┘
    
    
    THE CORE IDEA OF REST:
    ═════════════════════
    
    Everything is a RESOURCE.
    Each resource has a unique URL.
    You use HTTP methods to interact with resources.
    
    Resource         URL                    What it represents
    ────────         ───                    ──────────────────
    All users        /api/users             Collection of users
    Single user      /api/users/42          User with ID 42
    User's orders    /api/users/42/orders   Orders for user 42
    Single order     /api/orders/100        Order with ID 100
    All products     /api/products          Collection of products
```

---

## 🔧 HTTP Methods — The CRUD Operations

```
    ┌────────────┬─────────────┬────────────────────────────────────────────┐
    │  HTTP      │  CRUD       │  What It Does                            │
    │  Method    │  Operation  │                                          │
    ├────────────┼─────────────┼────────────────────────────────────────────┤
    │  GET       │  Read       │  Retrieve data. NEVER changes data.      │
    │            │             │  GET /api/products → list all products   │
    │            │             │  GET /api/products/42 → get product #42  │
    ├────────────┼─────────────┼────────────────────────────────────────────┤
    │  POST      │  Create     │  Create new data. Sends data in body.    │
    │            │             │  POST /api/products → create new product │
    ├────────────┼─────────────┼────────────────────────────────────────────┤
    │  PUT       │  Update     │  Replace entire resource.                 │
    │            │  (full)     │  PUT /api/products/42 → replace product  │
    ├────────────┼─────────────┼────────────────────────────────────────────┤
    │  PATCH     │  Update     │  Update part of a resource.              │
    │            │  (partial)  │  PATCH /api/products/42 → update price   │
    ├────────────┼─────────────┼────────────────────────────────────────────┤
    │  DELETE    │  Delete     │  Remove a resource.                       │
    │            │             │  DELETE /api/products/42 → delete product│
    └────────────┴─────────────┴────────────────────────────────────────────┘
    
    
    VISUAL FLOW:
    ═══════════
    
    Frontend                                  Backend (REST API)
       │                                           │
       │  GET /api/products                        │
       │──────────────────────────────────────────▶│  Read from DB
       │  ◀── 200 OK + [{...}, {...}, {...}]      │
       │                                           │
       │  POST /api/products                       │
       │  Body: {"name": "Mouse", "price": 999}   │
       │──────────────────────────────────────────▶│  Insert into DB
       │  ◀── 201 Created + {"id": 4, ...}        │
       │                                           │
       │  PUT /api/products/4                      │
       │  Body: {"name": "Mouse", "price": 1299}  │
       │──────────────────────────────────────────▶│  Update in DB
       │  ◀── 200 OK + {"id": 4, ...}             │
       │                                           │
       │  DELETE /api/products/4                   │
       │──────────────────────────────────────────▶│  Delete from DB
       │  ◀── 204 No Content                      │
```

---

## 📊 HTTP Status Codes — The API's Language

```
    Status codes tell the client WHAT HAPPENED:
    
    ┌─────────┬────────────────────────────────────────────────────────┐
    │  Range  │  Meaning                                              │
    ├─────────┼────────────────────────────────────────────────────────┤
    │  1xx    │  Informational (rarely used in REST)                  │
    │  2xx    │  ✅ SUCCESS — Everything worked!                      │
    │  3xx    │  ↗️ REDIRECT — Look somewhere else                    │
    │  4xx    │  ❌ CLIENT ERROR — You messed up                     │
    │  5xx    │  💥 SERVER ERROR — We messed up                      │
    └─────────┴────────────────────────────────────────────────────────┘
    
    MOST COMMON STATUS CODES:
    ┌───────┬──────────────────────┬────────────────────────────────────┐
    │ Code  │ Name                 │ When to Use                       │
    ├───────┼──────────────────────┼────────────────────────────────────┤
    │  200  │ OK                   │ GET/PUT/PATCH succeeded           │
    │  201  │ Created              │ POST succeeded, resource created  │
    │  204  │ No Content           │ DELETE succeeded, nothing to show │
    ├───────┼──────────────────────┼────────────────────────────────────┤
    │  301  │ Moved Permanently    │ URL changed forever               │
    │  304  │ Not Modified         │ Cached version is still good      │
    ├───────┼──────────────────────┼────────────────────────────────────┤
    │  400  │ Bad Request          │ Invalid data sent by client       │
    │  401  │ Unauthorized         │ Not logged in (no/invalid token)  │
    │  403  │ Forbidden            │ Logged in but not allowed         │
    │  404  │ Not Found            │ Resource doesn't exist            │
    │  409  │ Conflict             │ Duplicate data (e.g., same email) │
    │  422  │ Unprocessable Entity │ Validation error (missing field)  │
    │  429  │ Too Many Requests    │ Rate limited — slow down!         │
    ├───────┼──────────────────────┼────────────────────────────────────┤
    │  500  │ Internal Server Error│ Bug in your code / server crashed │
    │  502  │ Bad Gateway          │ Upstream server is down           │
    │  503  │ Service Unavailable  │ Server overloaded or in maint.   │
    │  504  │ Gateway Timeout      │ Upstream server too slow          │
    └───────┴──────────────────────┴────────────────────────────────────┘
```

---

## 🎯 REST URL Design — Best Practices

```
    ✅ GOOD REST URLs:                    ❌ BAD REST URLs:
    ═══════════════                       ════════════════
    
    GET  /api/users                       GET  /api/getUsers
    GET  /api/users/42                    GET  /api/getUserById?id=42
    POST /api/users                       POST /api/createUser
    PUT  /api/users/42                    POST /api/updateUser
    DELETE /api/users/42                  GET  /api/deleteUser?id=42
    
    GET  /api/users/42/orders             GET  /api/getUserOrders?userId=42
    GET  /api/orders?status=pending       GET  /api/getPendingOrders
    
    
    RULES:
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │  1. Use NOUNS, not verbs                                     │
    │     ✅ /api/products    ❌ /api/getProducts                  │
    │                                                              │
    │  2. Use PLURAL nouns                                         │
    │     ✅ /api/users       ❌ /api/user                         │
    │                                                              │
    │  3. Use LOWERCASE with hyphens                               │
    │     ✅ /api/order-items ❌ /api/OrderItems                   │
    │                                                              │
    │  4. Use PATH for hierarchy, QUERY for filtering              │
    │     ✅ /api/users/42/orders?status=shipped                   │
    │                                                              │
    │  5. Version your API                                         │
    │     ✅ /api/v1/products ✅ /api/v2/products                  │
    │                                                              │
    │  6. Don't put verbs in URLs                                  │
    │     ✅ POST /api/orders (create order)                       │
    │     ❌ POST /api/orders/create                               │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## 💻 Building a REST API — Complete Examples

### Python (Flask)

```python
"""
Complete REST API for a Product resource.
Demonstrates all CRUD operations with proper status codes.
"""
from flask import Flask, jsonify, request, abort

app = Flask(__name__)

# In-memory "database" (use PostgreSQL in production!)
products = [
    {"id": 1, "name": "Laptop", "price": 59999, "category": "electronics"},
    {"id": 2, "name": "Phone",  "price": 29999, "category": "electronics"},
    {"id": 3, "name": "Book",   "price": 499,   "category": "books"},
]
next_id = 4

# ─── GET all products (with optional filtering) ───
@app.route('/api/v1/products', methods=['GET'])
def list_products():
    category = request.args.get('category')  # ?category=electronics
    result = products
    if category:
        result = [p for p in products if p['category'] == category]
    return jsonify({"data": result, "count": len(result)}), 200

# ─── GET single product ───
@app.route('/api/v1/products/<int:pid>', methods=['GET'])
def get_product(pid):
    product = next((p for p in products if p['id'] == pid), None)
    if not product:
        return jsonify({"error": "Product not found"}), 404
    return jsonify(product), 200

# ─── POST create new product ───
@app.route('/api/v1/products', methods=['POST'])
def create_product():
    global next_id
    data = request.get_json()
    # Validate required fields
    if not data or 'name' not in data or 'price' not in data:
        return jsonify({"error": "name and price are required"}), 400
    
    new_product = {
        "id": next_id,
        "name": data['name'],
        "price": data['price'],
        "category": data.get('category', 'general')
    }
    products.append(new_product)
    next_id += 1
    return jsonify(new_product), 201  # 201 = Created!

# ─── PUT update entire product ───
@app.route('/api/v1/products/<int:pid>', methods=['PUT'])
def update_product(pid):
    product = next((p for p in products if p['id'] == pid), None)
    if not product:
        return jsonify({"error": "Product not found"}), 404
    data = request.get_json()
    product['name'] = data.get('name', product['name'])
    product['price'] = data.get('price', product['price'])
    product['category'] = data.get('category', product['category'])
    return jsonify(product), 200

# ─── DELETE remove product ───
@app.route('/api/v1/products/<int:pid>', methods=['DELETE'])
def delete_product(pid):
    global products
    products = [p for p in products if p['id'] != pid]
    return '', 204  # 204 = No Content (deleted successfully)

if __name__ == '__main__':
    app.run(port=8000, debug=True)
```

### Java (Spring Boot)

```java
/**
 * Complete REST API with Spring Boot.
 * Demonstrates proper HTTP methods, status codes, and validation.
 */
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // GET /api/v1/products?category=electronics
    @GetMapping
    public ResponseEntity<Map<String, Object>> listProducts(
            @RequestParam(required = false) String category) {
        List<Product> products = (category != null)
            ? productService.findByCategory(category)
            : productService.findAll();
        return ResponseEntity.ok(Map.of(
            "data", products, "count", products.size()
        ));
    }

    // GET /api/v1/products/42
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)                    // 200 OK
            .orElse(ResponseEntity.notFound().build()); // 404
    }

    // POST /api/v1/products
    @PostMapping
    public ResponseEntity<Product> createProduct(
            @Valid @RequestBody ProductRequest request) {
        Product created = productService.create(request);
        URI location = URI.create("/api/v1/products/" + created.getId());
        return ResponseEntity.created(location).body(created); // 201
    }

    // PUT /api/v1/products/42
    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductRequest request) {
        Product updated = productService.update(id, request);
        return ResponseEntity.ok(updated); // 200
    }

    // DELETE /api/v1/products/42
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build(); // 204
    }
}
```

---

## 📦 Request & Response Anatomy

```
    A COMPLETE REST API REQUEST:
    ════════════════════════════
    
    POST /api/v1/products HTTP/1.1          ← Method + URL + HTTP version
    Host: api.mystore.com                   ← Server address
    Content-Type: application/json           ← Body format
    Authorization: Bearer eyJhbGci...       ← Auth token
    Accept: application/json                 ← What format I want back
    
    {                                        ← Request Body (JSON)
      "name": "Wireless Mouse",
      "price": 1299,
      "category": "electronics"
    }
    
    
    A COMPLETE REST API RESPONSE:
    ═════════════════════════════
    
    HTTP/1.1 201 Created                     ← Status code
    Content-Type: application/json           ← Response format
    Location: /api/v1/products/5             ← URL of new resource
    X-Request-Id: abc-123-def                ← Tracking ID
    
    {                                        ← Response Body (JSON)
      "id": 5,
      "name": "Wireless Mouse",
      "price": 1299,
      "category": "electronics",
      "created_at": "2024-01-15T10:30:00Z"
    }
```

---

## 🔄 REST Principles — The 6 Constraints

```
    ┌──────────────────────┬───────────────────────────────────────────────┐
    │  Principle           │  What It Means                               │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  1. Client-Server    │  Frontend and backend are SEPARATE.          │
    │                      │  They communicate only through the API.      │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  2. Stateless        │  Server doesn't remember previous requests.  │
    │                      │  Each request carries ALL needed info        │
    │                      │  (auth token, session data, etc.)            │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  3. Cacheable        │  Responses can be cached (with headers).     │
    │                      │  GET responses are often cached.             │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  4. Uniform          │  Consistent URL patterns and HTTP methods.   │
    │     Interface        │  Every resource follows the same rules.      │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  5. Layered System   │  Client doesn't know if it's talking to     │
    │                      │  the actual server or a proxy/LB/cache.     │
    ├──────────────────────┼───────────────────────────────────────────────┤
    │  6. Code on Demand   │  Server can send executable code to client  │
    │     (optional)       │  (e.g., JavaScript). Rarely used.           │
    └──────────────────────┴───────────────────────────────────────────────┘
    
    
    STATELESS — The Most Important One:
    ════════════════════════════════════
    
    ❌ STATEFUL (bad for REST):
    Request 1: "I'm user John" → Server remembers "John"
    Request 2: "Show my orders" → Server knows it's John
    
    ✅ STATELESS (REST way):
    Request 1: "I'm user John, token=abc123, show products"
    Request 2: "I'm user John, token=abc123, show my orders"
    
    Every request carries its own identity!
    This is why we send auth tokens with EVERY request.
    
    WHY STATELESS MATTERS:
    • Any server can handle any request (great for load balancing!)
    • No session data to sync between servers
    • Easy to scale horizontally (add more servers)
    • If a server crashes, no user state is lost
```

---

## 📊 Pagination, Filtering & Sorting

```
    When you have thousands of products, you can't return ALL of them!
    
    PAGINATION:
    ═══════════
    GET /api/v1/products?page=1&limit=20     ← Page 1, 20 items
    GET /api/v1/products?page=2&limit=20     ← Page 2, 20 items
    
    Response:
    {
      "data": [...20 products...],
      "pagination": {
        "page": 1,
        "limit": 20,
        "total": 350,
        "total_pages": 18
      }
    }
    
    
    FILTERING:
    ══════════
    GET /api/v1/products?category=electronics&min_price=1000&max_price=50000
    
    
    SORTING:
    ════════
    GET /api/v1/products?sort=price&order=asc     ← Cheapest first
    GET /api/v1/products?sort=created_at&order=desc  ← Newest first
    
    
    COMBINED:
    ═════════
    GET /api/v1/products?category=electronics&sort=price&order=asc&page=1&limit=10
    
    Translation: "Give me the 10 cheapest electronics products"
```

---

## 🏢 Real-World Examples

```
    ┌─────────────────┬──────────────────────────────────────────────────┐
    │  Company        │  REST API Usage                                 │
    ├─────────────────┼──────────────────────────────────────────────────┤
    │  GitHub         │  REST API for repos, issues, PRs.               │
    │                 │  GET /repos/facebook/react/issues               │
    │                 │  One of the best-designed REST APIs.            │
    ├─────────────────┼──────────────────────────────────────────────────┤
    │  Twitter/X      │  REST API for tweets, users, timelines.        │
    │                 │  GET /2/tweets?ids=1234,5678                    │
    ├─────────────────┼──────────────────────────────────────────────────┤
    │  Stripe         │  REST API for payments.                         │
    │                 │  POST /v1/charges → charge a credit card       │
    │                 │  Considered the "gold standard" of API design. │
    ├─────────────────┼──────────────────────────────────────────────────┤
    │  Spotify        │  REST API for music data.                       │
    │                 │  GET /v1/tracks/{id} → get song details        │
    ├─────────────────┼──────────────────────────────────────────────────┤
    │  Amazon (AWS)   │  Almost all AWS services use REST APIs.        │
    │                 │  S3, DynamoDB, Lambda — all REST.              │
    └─────────────────┴──────────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using GET to modify data
       → GET /api/deleteUser?id=42  (DANGEROUS! Crawlers can trigger!)
       ✅ DELETE /api/users/42

    ❌ Returning 200 for errors
       → 200 OK + {"error": "Not found"}  (confusing!)
       ✅ 404 Not Found + {"error": "Product not found"}

    ❌ Not versioning your API
       → Changing /api/products breaks all existing clients!
       ✅ /api/v1/products → /api/v2/products

    ❌ Deeply nested URLs
       → /api/users/42/orders/100/items/5/reviews/3 (too deep!)
       ✅ /api/reviews/3 or /api/order-items/5/reviews

    ❌ Inconsistent naming
       → /api/Products, /api/get_users, /api/OrderList
       ✅ /api/products, /api/users, /api/orders (lowercase, plural)

    ❌ Not handling errors gracefully
       → Returning raw stack traces to the client (security risk!)
       ✅ Return clean error messages: {"error": "...", "code": "..."}

    ❌ Not using pagination
       → GET /api/products returns 1 million rows (crashes client!)
       ✅ Default limit=20, max limit=100
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. REST API = HTTP methods + URLs + JSON + status codes.           ║
║     It's a CONVENTION, not a protocol.                               ║
║                                                                      ║
║  2. Use the right HTTP method: GET (read), POST (create),          ║
║     PUT/PATCH (update), DELETE (delete).                             ║
║                                                                      ║
║  3. Use proper status codes: 200 (OK), 201 (Created),              ║
║     400 (Bad Request), 404 (Not Found), 500 (Server Error).         ║
║                                                                      ║
║  4. REST is STATELESS — every request carries its own auth token.   ║
║     This makes scaling easy (any server can handle any request).     ║
║                                                                      ║
║  5. URL design matters: use nouns (not verbs), plural,              ║
║     lowercase, and hyphens. Version your API (/api/v1/).            ║
║                                                                      ║
║  6. Always paginate list endpoints. Never return unbounded data.    ║
║                                                                      ║
║  7. REST is the most common API style, but not the only one.        ║
║     GraphQL and gRPC solve problems where REST falls short.         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

REST APIs are great but have a key limitation: the server decides what data to return, and the client gets ALL of it — even fields it doesn't need. What if the client could ask for **exactly** the data it wants? That's what GraphQL does. Next: [Chapter 3.3: GraphQL](./03-graphql.md).

---

[⬅️ Previous: What is a Backend Server](./01-what-is-backend-server.md) | [⬆️ Index](../../00-INDEX.md) | [Next: GraphQL ➡️](./03-graphql.md)
