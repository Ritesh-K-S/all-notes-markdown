# API Design for LLD

Designing clean, consistent, and intuitive APIs is a key part of Low Level Design. This covers REST API design conventions and internal API (method) design.

---

## 1. REST API Design Principles

### Resource Naming Conventions

| Rule | Good | Bad |
|------|------|-----|
| Use **nouns**, not verbs | `/users` | `/getUsers` |
| Use **plural** nouns | `/orders` | `/order` |
| Use **lowercase** | `/user-profiles` | `/UserProfiles` |
| Use **hyphens** for readability | `/order-items` | `/order_items` |
| **Nest** for relationships | `/users/{id}/orders` | `/getUserOrders` |

### HTTP Methods Mapping

| Method | CRUD Operation | Example | Idempotent |
|--------|---------------|---------|-----------|
| `GET` | Read | `GET /users/123` | Yes |
| `POST` | Create | `POST /users` | No |
| `PUT` | Full Update | `PUT /users/123` | Yes |
| `PATCH` | Partial Update | `PATCH /users/123` | Yes |
| `DELETE` | Delete | `DELETE /users/123` | Yes |

### Standard API Design for a Resource

```
GET    /api/v1/products              → List all products
GET    /api/v1/products/{id}         → Get product by ID
POST   /api/v1/products              → Create a product
PUT    /api/v1/products/{id}         → Replace a product
PATCH  /api/v1/products/{id}         → Partially update a product
DELETE /api/v1/products/{id}         → Delete a product

# Nested resources
GET    /api/v1/users/{userId}/orders         → Get user's orders
POST   /api/v1/users/{userId}/orders         → Create order for user
GET    /api/v1/orders/{orderId}/items         → Get items in an order
```

---

## 2. Request & Response Design

### Request Body (POST/PUT)

```json
// POST /api/v1/users
{
    "name": "Ritesh Singh",
    "email": "ritesh@example.com",
    "phone": "+91-9876543210",
    "address": {
        "street": "123 Main St",
        "city": "Delhi",
        "state": "Delhi",
        "zipCode": "110001"
    }
}
```

### Successful Response

```json
// 201 Created
{
    "id": "usr_abc123",
    "name": "Ritesh Singh",
    "email": "ritesh@example.com",
    "phone": "+91-9876543210",
    "createdAt": "2026-04-19T10:30:00Z"
}
```

### Error Response (Consistent Format)

```json
// 400 Bad Request
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid request body",
        "details": [
            {
                "field": "email",
                "message": "Email format is invalid"
            },
            {
                "field": "phone",
                "message": "Phone number is required"
            }
        ]
    },
    "timestamp": "2026-04-19T10:30:00Z",
    "path": "/api/v1/users"
}
```

---

## 3. HTTP Status Codes

### Success Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| `200 OK` | Success | GET, PUT, PATCH |
| `201 Created` | Resource created | POST |
| `204 No Content` | Success, no body | DELETE |

### Client Error Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| `400 Bad Request` | Invalid input | Validation errors |
| `401 Unauthorized` | Not authenticated | Missing/invalid token |
| `403 Forbidden` | Not authorized | Insufficient permissions |
| `404 Not Found` | Resource doesn't exist | Invalid ID |
| `409 Conflict` | Conflict with current state | Duplicate email, concurrent edit |
| `422 Unprocessable Entity` | Valid syntax, invalid semantics | Business rule violation |
| `429 Too Many Requests` | Rate limit exceeded | Throttling |

### Server Error Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| `500 Internal Server Error` | Unexpected server error | Unhandled exceptions |
| `502 Bad Gateway` | Upstream failure | Dependency down |
| `503 Service Unavailable` | Temporarily unavailable | Maintenance |

---

## 4. Pagination

### Offset-Based Pagination

```
GET /api/v1/products?page=2&size=20

Response:
{
    "data": [ ... ],
    "pagination": {
        "page": 2,
        "size": 20,
        "totalElements": 150,
        "totalPages": 8,
        "hasNext": true,
        "hasPrevious": true
    }
}
```

### Cursor-Based Pagination (Better for large datasets)

```
GET /api/v1/products?cursor=abc123&limit=20

Response:
{
    "data": [ ... ],
    "pagination": {
        "nextCursor": "def456",
        "previousCursor": "xyz789",
        "limit": 20,
        "hasMore": true
    }
}
```

### Java Implementation

```java
public class PagedResponse<T> {
    private List<T> data;
    private PaginationInfo pagination;

    // Constructor, getters
}

public class PaginationInfo {
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean hasNext;
    private boolean hasPrevious;
}
```

---

## 5. Filtering, Sorting & Searching

```
# Filtering
GET /api/v1/products?category=electronics&minPrice=100&maxPrice=500

# Sorting
GET /api/v1/products?sortBy=price&sortOrder=asc

# Multiple sort fields
GET /api/v1/products?sort=price,asc&sort=name,desc

# Searching
GET /api/v1/products?search=laptop

# Combined
GET /api/v1/products?category=electronics&minPrice=100&sort=price,asc&page=1&size=20
```

---

## 6. API Versioning

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL Path** | `/api/v1/users` | Simple, visible | URL pollution |
| **Header** | `Api-Version: 1` | Clean URL | Hidden |
| **Query Param** | `/users?version=1` | Simple | Messy |

URL path versioning is the most common and recommended approach.

---

## 7. Internal API Design (Method Design)

### Method Naming Conventions

```java
// Use verb + noun pattern
public User findUserById(String userId)         // Good
public User user(String id)                      // Bad — unclear

public List<Order> getOrdersByStatus(OrderStatus status)   // Good
public List<Order> orders(String s)                         // Bad

public void cancelOrder(String orderId)          // Good
public void process(String id)                   // Bad — too vague
```

### Method Design Principles

#### Keep Parameters Minimal (Max 3-4)

```java
// Bad — too many parameters
public Order createOrder(String userId, String productId, int quantity,
                         String shippingAddress, String paymentMethod,
                         String couponCode, boolean giftWrap) { }

// Good — use a request object
public Order createOrder(CreateOrderRequest request) { }

public class CreateOrderRequest {
    private String userId;
    private List<OrderItemRequest> items;
    private Address shippingAddress;
    private String paymentMethod;
    private String couponCode;
    private boolean giftWrap;
}
```

#### Return Meaningful Types

```java
// Bad — returns null on failure
public User findUser(String id) {
    // returns null if not found — caller might forget to check
}

// Good — use Optional
public Optional<User> findUser(String id) {
    // Caller is forced to handle absence
}

// Good — throw exception for required lookups
public User getUser(String id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

#### Command Query Separation (CQS)

```java
// Query — returns data, no side effects
public User getUserById(String id) { }
public List<Order> getRecentOrders(String userId) { }
public int getOrderCount(String userId) { }

// Command — performs action, returns void (or the result)
public void cancelOrder(String orderId) { }
public void updateUserEmail(String userId, String newEmail) { }
public Order placeOrder(CreateOrderRequest request) { }
```

---

## 8. DTO Pattern (Data Transfer Object)

Separate internal domain models from external API representation.

```java
// Domain Entity — internal
public class User {
    private String id;
    private String name;
    private String email;
    private String passwordHash;  // Should NEVER be exposed
    private boolean isDeleted;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Request DTO — what client sends
public class CreateUserRequest {
    private String name;
    private String email;
    private String password;  // Plain text — will be hashed
}

// Response DTO — what client receives
public class UserResponse {
    private String id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
    // No passwordHash, no isDeleted, no updatedAt
}

// Mapper — converts between domain and DTO
public class UserMapper {
    public static UserResponse toResponse(User user) {
        UserResponse response = new UserResponse();
        response.setId(user.getId());
        response.setName(user.getName());
        response.setEmail(user.getEmail());
        response.setCreatedAt(user.getCreatedAt());
        return response;
    }

    public static User toEntity(CreateUserRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        // Hash password before setting
        user.setPasswordHash(PasswordEncoder.encode(request.getPassword()));
        return user;
    }
}
```

---

## 9. API Design Checklist

- [ ] Resources named with plural nouns
- [ ] Proper HTTP methods used (GET/POST/PUT/PATCH/DELETE)
- [ ] Consistent error response format
- [ ] Appropriate HTTP status codes
- [ ] Pagination for list endpoints
- [ ] Filtering and sorting support
- [ ] API versioning in place
- [ ] DTOs used (no domain leakage)
- [ ] Sensitive data excluded from responses
- [ ] Request validation at the boundary
- [ ] Idempotency for PUT/DELETE operations
