# API Design Best Practices (REST, Pagination, Error Handling)

> **What you'll learn**: How to design APIs that developers love to use — covering URL structure, HTTP methods, pagination, filtering, error responses, versioning, and the conventions used by Stripe, GitHub, and Google that have become industry standards.

---

## Real-Life Analogy

Think of an API like a **restaurant menu**:

- The **menu categories** (appetizers, mains, desserts) are your **resource URLs** (`/users`, `/orders`, `/products`)
- The **actions** (order, modify, cancel) are your **HTTP methods** (POST, PUT, DELETE)
- The **prices and descriptions** are your **response format** (clear, predictable)
- **Special requests** ("no onions") are your **query parameters** (`?sort=name&filter=active`)
- When something goes wrong ("we're out of fish"), the waiter gives you a clear explanation — that's your **error response**

A great menu is organized, predictable, and tells you exactly what you'll get. A great API does the same thing.

---

## REST Fundamentals — Quick Recap

REST (Representational State Transfer) treats everything as a **resource** that you interact with using standard HTTP methods:

```
┌──────────┬──────────────────┬───────────────────────────────┐
│  Method  │     Purpose       │          Example              │
├──────────┼──────────────────┼───────────────────────────────┤
│  GET     │ Read a resource   │ GET /users/123                │
│  POST    │ Create a resource │ POST /users                   │
│  PUT     │ Replace resource  │ PUT /users/123                │
│  PATCH   │ Partial update    │ PATCH /users/123              │
│  DELETE  │ Delete resource   │ DELETE /users/123             │
└──────────┴──────────────────┴───────────────────────────────┘
```

---

## URL Design — The Foundation

### Rule 1: Use Nouns, Not Verbs

```
✅ GOOD (Resources as nouns):
───────────────────────────────
GET    /users              → List users
GET    /users/123          → Get user 123
POST   /users              → Create a user
PUT    /users/123          → Update user 123
DELETE /users/123          → Delete user 123

❌ BAD (Verbs in URLs):
───────────────────────────────
GET    /getUser/123        → NO!
POST   /createUser         → NO!
POST   /deleteUser/123     → NO!
GET    /fetchAllUsers      → NO!
```

### Rule 2: Use Plural Nouns

```
✅ /users/123         → Consistent: "get user 123 from the users collection"
❌ /user/123          → Inconsistent with /user (is this one user or all?)
```

### Rule 3: Nest Resources to Show Relationships

```
GET /users/123/orders          → Orders belonging to user 123
GET /users/123/orders/456      → Order 456 of user 123
GET /orders/456/items          → Items in order 456

BUT: Don't nest deeper than 2 levels!
❌ /users/123/orders/456/items/789/reviews  → Too deep!
✅ /items/789/reviews                       → Flatten it
```

### Rule 4: Use kebab-case for URLs

```
✅ /user-profiles/123
✅ /order-items
❌ /userProfiles/123    (camelCase)
❌ /user_profiles/123   (snake_case)
```

### Full URL Design Example:

```
┌─────────────────────────────────────────────────────────────┐
│            E-COMMERCE API URL STRUCTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Products:                                                    │
│    GET    /products                   List all products       │
│    GET    /products/:id               Get one product         │
│    POST   /products                   Create product          │
│    PATCH  /products/:id               Update product          │
│    DELETE /products/:id               Delete product          │
│    GET    /products/:id/reviews       Product's reviews       │
│                                                               │
│  Orders:                                                      │
│    GET    /orders                     List user's orders      │
│    POST   /orders                     Create order            │
│    GET    /orders/:id                 Get order details       │
│    POST   /orders/:id/cancel          Cancel order (action)  │
│    GET    /orders/:id/items           Order items             │
│                                                               │
│  Users:                                                       │
│    GET    /users/me                   Current user profile    │
│    PATCH  /users/me                   Update profile          │
│    GET    /users/:id/addresses        User's addresses        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## HTTP Status Codes — Tell the Client What Happened

```
┌─────────────────────────────────────────────────────────────┐
│                 STATUS CODE CHEAT SHEET                       │
├─────────┬───────────────────────────────────────────────────┤
│  Code   │  Meaning                                           │
├─────────┼───────────────────────────────────────────────────┤
│         │  SUCCESS                                           │
│  200    │  OK — Request succeeded                            │
│  201    │  Created — Resource created (POST success)         │
│  204    │  No Content — Success, nothing to return (DELETE)  │
│         │                                                    │
│         │  CLIENT ERRORS (your fault)                        │
│  400    │  Bad Request — Invalid input/validation failed     │
│  401    │  Unauthorized — Not authenticated                  │
│  403    │  Forbidden — Authenticated but not allowed         │
│  404    │  Not Found — Resource doesn't exist                │
│  409    │  Conflict — Resource conflict (duplicate email)    │
│  422    │  Unprocessable — Valid JSON but semantic error      │
│  429    │  Too Many Requests — Rate limit exceeded           │
│         │                                                    │
│         │  SERVER ERRORS (our fault)                         │
│  500    │  Internal Server Error — Bug in our code           │
│  502    │  Bad Gateway — Upstream service down               │
│  503    │  Service Unavailable — Overloaded/maintenance      │
│  504    │  Gateway Timeout — Upstream too slow               │
└─────────┴───────────────────────────────────────────────────┘
```

---

## Error Response Design

### The Problem with Bad Error Responses:

```json
// ❌ BAD: No useful information
{"error": "Something went wrong"}

// ❌ BAD: Exposes internal details
{"error": "NullPointerException at UserService.java:142"}

// ❌ BAD: No machine-readable code
{"message": "The email you provided is already in use"}
```

### The Standard Error Response Format (RFC 7807):

```json
// ✅ GOOD: Structured, actionable error response
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "Validation Failed",
    "status": 422,
    "detail": "The request body contains invalid fields",
    "instance": "/users/signup",
    "timestamp": "2024-01-15T10:30:00Z",
    "errors": [
        {
            "field": "email",
            "code": "DUPLICATE",
            "message": "This email is already registered"
        },
        {
            "field": "password",
            "code": "TOO_SHORT",
            "message": "Password must be at least 8 characters"
        }
    ]
}
```

### Error Response Flow:

```
Client Request ──▶ Server validates ──▶ Error found?
                                            │
                   ┌────────────────────────┘
                   ▼
    ┌────────────────────────────────────┐
    │  HTTP Status: 422                   │
    │  Content-Type: application/json     │
    │                                      │
    │  Body:                               │
    │  {                                   │
    │    "type": "validation-error",       │
    │    "title": "Validation Failed",     │
    │    "errors": [...]                   │
    │  }                                   │
    └────────────────────────────────────┘
```

---

## Pagination — Handling Large Collections

### Why Paginate?

```
GET /products → Returns ALL 2 million products? ← CATASTROPHE!
                Server dies. Client dies. Everyone dies.

GET /products?page=1&size=20 → Returns 20 products ← Perfect!
```

### Offset-Based Pagination (Simple)

```
GET /products?page=2&size=20

Response:
{
    "data": [...20 items...],
    "pagination": {
        "page": 2,
        "size": 20,
        "totalItems": 1543,
        "totalPages": 78,
        "hasNext": true,
        "hasPrevious": true
    }
}
```

**Problem**: Offset pagination is slow for large offsets. `OFFSET 1000000` means the DB scans and discards a million rows!

### Cursor-Based Pagination (Better for Large Datasets)

```
GET /products?limit=20&cursor=eyJpZCI6MTAwfQ==

Response:
{
    "data": [...20 items...],
    "pagination": {
        "limit": 20,
        "nextCursor": "eyJpZCI6MTIwfQ==",
        "previousCursor": "eyJpZCI6OTl9",
        "hasMore": true
    }
}
```

```
Offset Pagination:                 Cursor Pagination:
─────────────────────             ──────────────────────
Page 1: Skip 0, Take 20          Start: id > 0, Take 20
Page 2: Skip 20, Take 20         Next: id > 20, Take 20
Page 3: Skip 40, Take 20         Next: id > 40, Take 20
...                               ...
Page 5000: Skip 100000 😱        Next: id > 99980 ✅ (fast!)
    ↑ DB scans 100K rows              ↑ Uses index directly
```

### Comparison:

| Feature | Offset | Cursor |
|---------|--------|--------|
| Jump to page N | ✅ Yes | ❌ No (sequential only) |
| Performance at page 1000 | ❌ Slow | ✅ Fast |
| Consistent with real-time inserts | ❌ Items shift | ✅ Stable |
| Implementation complexity | ✅ Simple | ⚠️ Moderate |
| Best for | Small datasets, UI page numbers | Infinite scroll, large datasets |

---

## Filtering, Sorting, and Searching

```
┌─────────────────────────────────────────────────────────────┐
│              QUERY PARAMETER CONVENTIONS                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Filtering:                                                   │
│    GET /products?category=electronics&price_min=100            │
│    GET /orders?status=shipped&created_after=2024-01-01         │
│                                                               │
│  Sorting:                                                     │
│    GET /products?sort=price          (ascending, default)     │
│    GET /products?sort=-price         (descending, - prefix)   │
│    GET /products?sort=category,-price (multi-sort)            │
│                                                               │
│  Searching:                                                   │
│    GET /products?q=wireless+headphones                        │
│    GET /users?search=john                                     │
│                                                               │
│  Field Selection (Sparse Fieldsets):                          │
│    GET /users/123?fields=name,email,avatar                    │
│    (Only return these fields — reduces payload)               │
│                                                               │
│  Including Relations:                                         │
│    GET /orders/123?include=items,customer                     │
│    (Embed related resources to avoid N+1 calls)               │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Request and Response Design

### Request Body Conventions:

```json
// POST /orders - Create an order
{
    "customer_id": "cust_abc123",
    "items": [
        {
            "product_id": "prod_xyz",
            "quantity": 2
        }
    ],
    "shipping_address": {
        "street": "123 Main St",
        "city": "Mumbai",
        "zip": "400001",
        "country": "IN"
    }
}
```

### Response Envelope Pattern:

```json
// Successful response
{
    "data": {
        "id": "ord_123",
        "status": "created",
        "total": 2499.00,
        "currency": "INR",
        "created_at": "2024-01-15T10:30:00Z"
    },
    "meta": {
        "request_id": "req_abc123"
    }
}

// List response
{
    "data": [...items...],
    "pagination": {
        "page": 1,
        "total": 142
    },
    "meta": {
        "request_id": "req_def456"
    }
}
```

### Naming Conventions:

```
┌───────────────────────────────────────────────────────┐
│  Convention          │  Example        │  Used By      │
├──────────────────────┼─────────────────┼──────────────┤
│  snake_case          │  created_at     │  Stripe, GH  │
│  camelCase           │  createdAt      │  Google       │
│  Consistent dates    │  ISO 8601       │  Everyone     │
│  IDs with prefix     │  usr_abc123     │  Stripe       │
│  Monetary amounts    │  cents (int)    │  Stripe       │
└───────────────────────────────────────────────────────┘

Pick ONE convention and stick with it across your entire API!
```

---

## How It Works Internally — Request Processing Pipeline

```
Client Request
    │
    ▼
┌────────────────────┐
│   API Gateway       │  Rate limiting, auth, routing
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Middleware        │  Request ID, logging, CORS
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Input Validation  │  Schema validation (400 if invalid)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Authentication    │  Verify token (401 if invalid)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Authorization     │  Check permissions (403 if denied)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Business Logic    │  Execute the actual operation
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Response Format   │  Serialize to JSON, set headers
└─────────┬──────────┘
          │
          ▼
    Response sent
```

---

## Code Examples

### Python: Well-Designed REST API with Flask

```python
from flask import Flask, request, jsonify
from functools import wraps
import uuid
from datetime import datetime

app = Flask(__name__)

# ─── Error handler ───────────────────────────────────────
@app.errorhandler(422)
def validation_error(e):
    return jsonify({
        "type": "validation-error",
        "title": "Validation Failed",
        "status": 422,
        "detail": str(e.description),
        "errors": e.description.get("errors", []) if isinstance(e.description, dict) else []
    }), 422

# ─── Pagination helper ───────────────────────────────────
def paginate(query, page, size):
    total = query.count()
    items = query.offset((page - 1) * size).limit(size).all()
    return {
        "data": [item.to_dict() for item in items],
        "pagination": {
            "page": page,
            "size": size,
            "total_items": total,
            "total_pages": (total + size - 1) // size,
            "has_next": page * size < total,
            "has_previous": page > 1
        }
    }

# ─── Products endpoint with filtering, sorting, pagination ─
@app.route("/api/v1/products", methods=["GET"])
def list_products():
    # Pagination params
    page = request.args.get("page", 1, type=int)
    size = request.args.get("size", 20, type=int)
    size = min(size, 100)  # Cap at 100 to prevent abuse
    
    # Filtering
    category = request.args.get("category")
    price_min = request.args.get("price_min", type=float)
    price_max = request.args.get("price_max", type=float)
    
    # Sorting
    sort = request.args.get("sort", "created_at")
    
    # Build query
    query = Product.query
    if category:
        query = query.filter(Product.category == category)
    if price_min:
        query = query.filter(Product.price >= price_min)
    if price_max:
        query = query.filter(Product.price <= price_max)
    
    # Apply sort (- prefix = descending)
    if sort.startswith("-"):
        query = query.order_by(getattr(Product, sort[1:]).desc())
    else:
        query = query.order_by(getattr(Product, sort).asc())
    
    return jsonify(paginate(query, page, size))

# ─── Create product with validation ─────────────────────
@app.route("/api/v1/products", methods=["POST"])
def create_product():
    data = request.get_json()
    
    # Validate required fields
    errors = []
    if not data.get("name"):
        errors.append({"field": "name", "code": "REQUIRED", 
                       "message": "Product name is required"})
    if not data.get("price") or data["price"] <= 0:
        errors.append({"field": "price", "code": "INVALID",
                       "message": "Price must be a positive number"})
    
    if errors:
        return jsonify({
            "type": "validation-error",
            "title": "Validation Failed",
            "status": 422,
            "errors": errors
        }), 422
    
    product = Product(
        id=f"prod_{uuid.uuid4().hex[:12]}",
        name=data["name"],
        price=data["price"],
        category=data.get("category", "general"),
        created_at=datetime.utcnow()
    )
    db.session.add(product)
    db.session.commit()
    
    return jsonify({"data": product.to_dict()}), 201
```

### Java: Well-Designed REST API with Spring Boot

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {
    
    private final ProductService productService;
    
    // ─── List with filtering, sorting, pagination ──────────
    @GetMapping
    public ResponseEntity<PagedResponse<ProductDTO>> listProducts(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal priceMin,
            @RequestParam(required = false) BigDecimal priceMax,
            @RequestParam(defaultValue = "created_at") String sort) {
        
        // Cap page size
        size = Math.min(size, 100);
        
        ProductFilter filter = ProductFilter.builder()
            .category(category)
            .priceMin(priceMin)
            .priceMax(priceMax)
            .build();
        
        PagedResponse<ProductDTO> response = productService.findAll(
            filter, page, size, sort);
        
        return ResponseEntity.ok(response);
    }
    
    // ─── Create with validation ────────────────────────────
    @PostMapping
    public ResponseEntity<ApiResponse<ProductDTO>> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        
        ProductDTO product = productService.create(request);
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.of(product));
    }
    
    // ─── Get single resource ───────────────────────────────
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductDTO>> getProduct(
            @PathVariable String id) {
        
        return productService.findById(id)
            .map(p -> ResponseEntity.ok(ApiResponse.of(p)))
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }
    
    // ─── Global error handler ──────────────────────────────
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = ErrorResponse.builder()
            .type("resource-not-found")
            .title("Resource Not Found")
            .status(404)
            .detail(ex.getMessage())
            .build();
        return ResponseEntity.status(404).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult()
            .getFieldErrors().stream()
            .map(f -> new FieldError(f.getField(), "INVALID", 
                f.getDefaultMessage()))
            .toList();
        
        ErrorResponse error = ErrorResponse.builder()
            .type("validation-error")
            .title("Validation Failed")
            .status(422)
            .errors(errors)
            .build();
        return ResponseEntity.status(422).body(error);
    }
}

// ─── Response wrapper classes ──────────────────────────────
public record ApiResponse<T>(T data, Meta meta) {
    public static <T> ApiResponse<T> of(T data) {
        return new ApiResponse<>(data, new Meta(UUID.randomUUID().toString()));
    }
}

public record PagedResponse<T>(
    List<T> data, 
    Pagination pagination
) {}

public record Pagination(
    int page, int size, long totalItems, 
    int totalPages, boolean hasNext, boolean hasPrevious
) {}
```

---

## API Versioning Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                   VERSIONING APPROACHES                       │
├──────────────┬──────────────────────────────────────────────┤
│  Strategy    │  Example                                      │
├──────────────┼──────────────────────────────────────────────┤
│  URL Path    │  GET /api/v1/users     ← Most common         │
│  (Stripe)    │  GET /api/v2/users                           │
│              │                                               │
│  Header      │  GET /api/users                              │
│  (GitHub)    │  Accept: application/vnd.github.v3+json      │
│              │                                               │
│  Query Param │  GET /api/users?version=2                    │
│              │  (Rarely used, not recommended)               │
│              │                                               │
│  Date-based  │  Stripe-Version: 2024-01-15                  │
│  (Stripe)    │  (Version by release date)                   │
└──────────────┴──────────────────────────────────────────────┘

RECOMMENDATION: URL path versioning (/api/v1/) for simplicity.
Only bump the version for BREAKING changes.
```

---

## Real-World Example

### Stripe's API (The Gold Standard)

Stripe is widely considered to have the best-designed API in the industry:

```
┌─────────────────────────────────────────────────────────────┐
│  STRIPE API DESIGN PRINCIPLES                                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Prefixed IDs:       ch_abc123 (charge), cus_xyz (cust)  │
│  2. Idempotency:        Idempotency-Key header              │
│  3. Expandable fields:  ?expand[]=customer                   │
│  4. Consistent errors:  { type, code, message, param }      │
│  5. Date versioning:    Stripe-Version: 2024-01-15          │
│  6. Cursor pagination:  ?starting_after=ch_abc123           │
│  7. Amounts in cents:   amount: 2000 (means $20.00)         │
│  8. Webhook events:     Structured event objects            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### GitHub's API

- Uses `Link` headers for pagination: `Link: <...?page=3>; rel="next"`
- Rich error messages with documentation URLs
- Rate limit info in headers: `X-RateLimit-Remaining: 42`
- Supports both REST and GraphQL

### Google's API Design Guide

Google published an open [API Design Guide](https://cloud.google.com/apis/design) that recommends:
- Resource-oriented design
- Standard methods (List, Get, Create, Update, Delete)
- Custom methods when standard ones don't fit: `POST /orders/123:cancel`

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using verbs in URLs | Breaks REST convention; confusing | Use nouns: `POST /orders` not `POST /createOrder` |
| Returning 200 for errors | Client thinks it succeeded | Use appropriate 4xx/5xx codes |
| Inconsistent naming | `camelCase` here, `snake_case` there | Pick one convention for the entire API |
| No pagination | API returns 10 million records | Always paginate collections |
| Exposing database IDs | Sequential IDs reveal business info | Use UUIDs or prefixed IDs |
| No rate limiting | One client kills your API | Implement rate limits from day one |
| Breaking changes without versioning | All clients break simultaneously | Version your API; deprecate gradually |
| Nested URLs too deep | `/a/1/b/2/c/3/d/4` is unmaintainable | Max 2 levels of nesting |
| No request ID | Can't trace bugs across services | Include `request_id` in every response |
| Returning internal errors to client | Security risk + bad UX | Generic 500 message + log details server-side |

---

## When to Use / When NOT to Use

### Use REST APIs When:
- Building public APIs consumed by third parties
- CRUD operations dominate (create, read, update, delete)
- Clients need caching (HTTP caching works naturally with REST)
- You want simplicity and broad tool support
- Multiple client types (web, mobile, IoT)

### Consider Alternatives When:
- Complex queries with many joins → **GraphQL** (Chapter 3.3)
- Real-time data → **WebSockets** or **SSE** (Chapter 3.5, 3.6)
- Internal service-to-service → **gRPC** for performance (Chapter 3.4)
- Event-driven workflows → **Message queues** (Chapter 11)
- Bulk data transfer → Streaming APIs or file downloads

---

## Key Takeaways

1. **Consistency is king** — Pick conventions (naming, pagination, errors) and apply them uniformly across your ENTIRE API. Inconsistency confuses developers more than bad choices.

2. **Design for the consumer** — Your API exists to serve clients. Make common operations easy. Provide good defaults. Don't make clients jump through hoops.

3. **Error responses are part of the API** — Spend as much time designing error responses as success responses. Include machine-readable codes, human-readable messages, and field-level details.

4. **Paginate everything** — Never return unbounded collections. Default to 20 items. Cap at 100. Use cursor-based pagination for large/real-time datasets.

5. **Version from day one** — Put `/v1/` in your URL from the start. It costs nothing and saves a painful migration later.

6. **Idempotency saves lives** — Make operations safe to retry. Use idempotency keys for payment-like operations (see Chapter 12.7).

7. **Study the best** — Stripe, GitHub, and Twilio APIs are masterclasses in API design. Read their documentation for inspiration before designing your own.

---

## What's Next?

Next, we'll explore the **Saga Pattern** — how to manage distributed transactions across multiple microservices when you can't use a single database transaction anymore.

See: [04-saga-pattern.md](./04-saga-pattern.md)
