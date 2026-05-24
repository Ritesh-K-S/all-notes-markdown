# Chapter 3.3: GraphQL — Flexible Queries from the Client

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: How GraphQL lets clients ask for exactly the data they need — no more, no less — solving the over-fetching and under-fetching problems of REST APIs.

---

## 🧠 Real-Life Analogy: Buffet vs Custom Order

**REST API** = A set menu. You order "Combo #3" and get burger + fries + drink — even if you only wanted the burger. You get everything or nothing.

**GraphQL** = A custom kitchen. You say: "I want a burger with extra cheese, no pickles, and a small drink." You get **exactly** what you asked for.

```
    REST API (Fixed Response):
    ══════════════════════════
    
    Client: "Give me User #42"
    Server: Here's EVERYTHING about User #42:
    {
      "id": 42,
      "name": "Ritesh",
      "email": "ritesh@example.com",
      "phone": "+91-9876543210",       ← You didn't need this!
      "address": "Mumbai, India",      ← Or this!
      "avatar_url": "...",             ← Or this!
      "created_at": "2020-01-01",      ← Or this!
      "preferences": {...}             ← Definitely not this!
    }
    
    
    GraphQL (Custom Response):
    ══════════════════════════
    
    Client: "Give me User #42's name and email ONLY"
    query {
      user(id: 42) {
        name
        email
      }
    }
    
    Server: Here's EXACTLY what you asked for:
    {
      "name": "Ritesh",
      "email": "ritesh@example.com"
    }
    
    Nothing extra! 🎯
```

---

## 📖 The Problems GraphQL Solves

### Problem 1: Over-fetching

```
    REST: GET /api/users/42
    Returns 30 fields, but mobile app only needs name + avatar.
    
    ┌──────────────────────────────────────────────────────┐
    │  REST Response: 2 KB                                 │
    │  ████████████████████████████████████████████████     │
    │  ██ Used ██████████████████ Wasted ██████████████     │
    │  (200 bytes)              (1800 bytes wasted!)       │
    │                                                      │
    │  GraphQL Response: 200 bytes                         │
    │  ██ Used ██                                          │
    │  (exactly what was needed!)                          │
    └──────────────────────────────────────────────────────┘
    
    On mobile with slow 3G: this MATTERS!
```

### Problem 2: Under-fetching (N+1 requests)

```
    REST: To show a user's profile with their posts and comments:
    
    Request 1: GET /api/users/42              → Get user
    Request 2: GET /api/users/42/posts        → Get posts
    Request 3: GET /api/posts/101/comments    → Get comments for post 1
    Request 4: GET /api/posts/102/comments    → Get comments for post 2
    Request 5: GET /api/posts/103/comments    → Get comments for post 3
    
    5 HTTP requests! Each with latency overhead! 🐌
    
    
    GraphQL: ONE request gets EVERYTHING:
    
    query {
      user(id: 42) {
        name
        avatar
        posts {
          title
          comments {
            text
            author { name }
          }
        }
      }
    }
    
    1 HTTP request! All data in one response! ⚡
```

---

## 🔧 How GraphQL Works

```
    GraphQL Architecture:
    ═════════════════════
    
    ┌──────────┐    POST /graphql     ┌──────────────────────┐
    │  Client  │ ───────────────────▶ │  GraphQL Server      │
    │          │                      │                      │
    │  Sends   │    JSON response     │  1. Parse query      │
    │  a QUERY │ ◀─────────────────── │  2. Validate against │
    │          │                      │     schema           │
    └──────────┘                      │  3. Execute resolvers│
                                      │  4. Return exact data│
                                      └──────────┬───────────┘
                                                 │
                                      Resolvers fetch from:
                                      ┌──────────┼──────────┐
                                      │          │          │
                                      ▼          ▼          ▼
                                  Database    REST API    Cache
                                  (SQL)       (external)  (Redis)
    
    
    KEY DIFFERENCE FROM REST:
    ═══════════════════════
    
    REST:    Multiple endpoints, server decides response shape
    ┌──────────────────────────────────────────┐
    │  GET  /api/users                         │
    │  GET  /api/users/42                      │
    │  GET  /api/users/42/posts                │
    │  POST /api/users                         │
    │  GET  /api/posts                         │
    │  GET  /api/posts/101/comments            │
    │  ... (many endpoints)                    │
    └──────────────────────────────────────────┘
    
    GraphQL: ONE endpoint, client decides response shape
    ┌──────────────────────────────────────────┐
    │  POST /graphql  (that's it! ONE endpoint)│
    │                                          │
    │  Client sends different QUERIES to get   │
    │  different data from the SAME endpoint.  │
    └──────────────────────────────────────────┘
```

---

## 📝 GraphQL Schema — The Type System

```
    The SCHEMA defines what data is available and how it's structured.
    Think of it as the "menu" for your API.
    
    ┌──────────────────────────────────────────────────────────────┐
    │  GraphQL Schema (SDL — Schema Definition Language):         │
    │                                                              │
    │  type User {                                                 │
    │    id: ID!              ← ! means required (non-null)       │
    │    name: String!                                             │
    │    email: String!                                            │
    │    age: Int                                                  │
    │    posts: [Post!]!      ← Array of Post objects             │
    │  }                                                          │
    │                                                              │
    │  type Post {                                                 │
    │    id: ID!                                                   │
    │    title: String!                                            │
    │    content: String                                           │
    │    author: User!        ← Relationship to User              │
    │    comments: [Comment!]                                      │
    │  }                                                          │
    │                                                              │
    │  type Comment {                                              │
    │    id: ID!                                                   │
    │    text: String!                                             │
    │    author: User!                                             │
    │  }                                                          │
    │                                                              │
    │  type Query {            ← What clients can READ            │
    │    user(id: ID!): User                                      │
    │    users: [User!]!                                           │
    │    post(id: ID!): Post                                      │
    │  }                                                          │
    │                                                              │
    │  type Mutation {         ← What clients can WRITE           │
    │    createUser(name: String!, email: String!): User!         │
    │    createPost(title: String!, authorId: ID!): Post!         │
    │    deletePost(id: ID!): Boolean!                            │
    │  }                                                          │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

---

## 📖 Three Operations: Query, Mutation, Subscription

```
    ┌──────────────┬────────────────────────────────────────────────┐
    │  Operation   │  Purpose                                      │
    ├──────────────┼────────────────────────────────────────────────┤
    │  Query       │  READ data (like GET in REST)                 │
    │              │  query { users { name email } }               │
    ├──────────────┼────────────────────────────────────────────────┤
    │  Mutation    │  WRITE data (like POST/PUT/DELETE in REST)    │
    │              │  mutation { createUser(name: "John") { id } } │
    ├──────────────┼────────────────────────────────────────────────┤
    │  Subscription│  REAL-TIME updates (like WebSocket)           │
    │              │  subscription { newMessage { text from } }    │
    └──────────────┴────────────────────────────────────────────────┘
```

### Query Examples

```graphql
# Simple query — get user's name and email
query {
  user(id: "42") {
    name
    email
  }
}

# Nested query — get user with their posts and post comments
query {
  user(id: "42") {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}

# Query with arguments — filter and pagination
query {
  products(category: "electronics", limit: 10, offset: 0) {
    name
    price
    inStock
  }
}

# Mutation — create a new product
mutation {
  createProduct(input: {
    name: "Wireless Mouse"
    price: 1299
    category: "electronics"
  }) {
    id
    name
    price
  }
}
```

---

## 💻 Code Examples

### Python (Strawberry GraphQL + Flask)

```python
"""
GraphQL API server using Strawberry (modern Python GraphQL library).
"""
import strawberry
from flask import Flask
from strawberry.flask.views import GraphQLView
from typing import List, Optional

# ─── Define Types (Schema) ───
@strawberry.type
class Product:
    id: int
    name: str
    price: int
    category: str

@strawberry.type
class User:
    id: int
    name: str
    email: str

# ─── Simulated database ───
products_db = [
    Product(id=1, name="Laptop", price=59999, category="electronics"),
    Product(id=2, name="Phone", price=29999, category="electronics"),
    Product(id=3, name="Book", price=499, category="books"),
]

# ─── Resolvers: Functions that fetch data ───
@strawberry.type
class Query:
    @strawberry.field
    def products(self, category: Optional[str] = None) -> List[Product]:
        """Resolver: fetches products, optionally filtered by category"""
        if category:
            return [p for p in products_db if p.category == category]
        return products_db
    
    @strawberry.field
    def product(self, id: int) -> Optional[Product]:
        """Resolver: fetches a single product by ID"""
        return next((p for p in products_db if p.id == id), None)

@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_product(self, name: str, price: int, category: str) -> Product:
        """Mutation: creates a new product"""
        new_id = max(p.id for p in products_db) + 1
        product = Product(id=new_id, name=name, price=price, category=category)
        products_db.append(product)
        return product

# ─── Create GraphQL schema and Flask app ───
schema = strawberry.Schema(query=Query, mutation=Mutation)
app = Flask(__name__)
app.add_url_rule("/graphql", view_func=GraphQLView.as_view("graphql", schema=schema))

if __name__ == "__main__":
    app.run(port=8000)
```

### Java (Spring Boot + GraphQL)

```java
/**
 * GraphQL API with Spring Boot + Spring GraphQL.
 * Schema defined in resources/graphql/schema.graphqls
 */

// schema.graphqls (in src/main/resources/graphql/)
// type Product {
//   id: ID!
//   name: String!
//   price: Int!
//   category: String!
// }
// type Query {
//   products(category: String): [Product!]!
//   product(id: ID!): Product
// }
// type Mutation {
//   createProduct(name: String!, price: Int!, category: String!): Product!
// }

@Controller
public class ProductGraphQLController {

    private final ProductRepository productRepo;

    public ProductGraphQLController(ProductRepository productRepo) {
        this.productRepo = productRepo;
    }

    // Query resolver: fetches products
    @QueryMapping
    public List<Product> products(@Argument String category) {
        if (category != null) {
            return productRepo.findByCategory(category);
        }
        return productRepo.findAll();
    }

    // Query resolver: fetches single product
    @QueryMapping
    public Optional<Product> product(@Argument Long id) {
        return productRepo.findById(id);
    }

    // Mutation resolver: creates product
    @MutationMapping
    public Product createProduct(
            @Argument String name,
            @Argument int price,
            @Argument String category) {
        Product product = new Product(name, price, category);
        return productRepo.save(product);
    }
}
```

---

## 📊 REST vs GraphQL — Complete Comparison

```
    ┌─────────────────────┬──────────────────────┬──────────────────────┐
    │  Aspect             │  REST                │  GraphQL             │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Endpoints          │  Many (/users,       │  ONE (/graphql)      │
    │                     │  /posts, /comments)  │                      │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Data shape         │  Server decides      │  Client decides      │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Over-fetching      │  Common problem      │  Solved ✅           │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Under-fetching     │  Common (N+1 calls)  │  Solved ✅           │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  HTTP methods       │  GET, POST, PUT, etc.│  POST only           │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Caching            │  Easy (HTTP cache)   │  Harder (need tools) │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Learning curve     │  Simple              │  Steeper             │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  File uploads       │  Easy (multipart)    │  Awkward             │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Error handling     │  HTTP status codes   │  Always 200, errors  │
    │                     │                      │  in response body    │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Type safety        │  Optional (OpenAPI)  │  Built-in (schema)   │
    ├─────────────────────┼──────────────────────┼──────────────────────┤
    │  Best for           │  Simple CRUD, public │  Complex UIs, mobile │
    │                     │  APIs, microservices │  apps, multiple      │
    │                     │                      │  client types        │
    └─────────────────────┴──────────────────────┴──────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using GraphQL for simple CRUD APIs
       → Overkill! REST is simpler for basic operations.
       ✅ Use GraphQL when you have complex, nested data needs.
    
    ❌ No query depth limiting
       → Client can send: { user { posts { comments { author { posts { ... } } } } } }
         This creates an infinitely deep query that crashes your server!
       ✅ Set max query depth (e.g., max depth = 5)
    
    ❌ N+1 problem in resolvers
       → Each post resolver fires a separate DB query for its author!
       ✅ Use DataLoader to batch queries (fetch all authors in ONE query)
    
    ❌ Exposing your entire database schema
       → GraphQL schema ≠ database schema. Don't mirror DB tables!
       ✅ Design your schema around client needs, not database structure.
    
    ❌ Not rate limiting GraphQL
       → One complex query can be as expensive as 100 REST calls!
       ✅ Implement query complexity analysis and cost limits.
```

---

## 🏢 Real-World Examples

```
    ┌───────────────┬───────────────────────────────────────────────────┐
    │  Company      │  Why They Use GraphQL                            │
    ├───────────────┼───────────────────────────────────────────────────┤
    │  Facebook     │  INVENTED GraphQL in 2012 for their mobile app. │
    │               │  Mobile needed precise data — no over-fetching.  │
    ├───────────────┼───────────────────────────────────────────────────┤
    │  GitHub       │  API v4 is GraphQL. Developers query exactly    │
    │               │  the repo/issue data they need.                  │
    ├───────────────┼───────────────────────────────────────────────────┤
    │  Shopify      │  Storefront API is GraphQL. Merchants build     │
    │               │  custom storefronts with flexible queries.       │
    ├───────────────┼───────────────────────────────────────────────────┤
    │  Netflix      │  Uses GraphQL Federation to compose data from   │
    │               │  hundreds of microservices into one API.         │
    ├───────────────┼───────────────────────────────────────────────────┤
    │  Pinterest    │  Switched from REST to GraphQL. Reduced API     │
    │               │  calls by ~60% on mobile.                        │
    └───────────────┴───────────────────────────────────────────────────┘
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. GraphQL = single endpoint where clients request EXACTLY the     ║
║     fields they need. No over-fetching, no under-fetching.          ║
║                                                                      ║
║  2. Three operations: Query (read), Mutation (write),               ║
║     Subscription (real-time).                                        ║
║                                                                      ║
║  3. Schema-first: the type system defines all available data,       ║
║     providing auto-documentation and type safety.                    ║
║                                                                      ║
║  4. Best suited for complex UIs with nested data, mobile apps       ║
║     (bandwidth matters), and apps with multiple client types.        ║
║                                                                      ║
║  5. GraphQL doesn't replace REST — use REST for simple CRUD,        ║
║     GraphQL for complex, client-driven data needs.                   ║
║                                                                      ║
║  6. Watch out for: query depth attacks, N+1 problems (use           ║
║     DataLoader), and missing rate limiting.                          ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

Both REST and GraphQL are great for frontend-to-backend communication. But when backend services need to talk to **each other** at high speed and low latency, they need something faster. Next: [Chapter 3.4: gRPC](./04-grpc.md) — the high-performance protocol for service-to-service communication.

---

[⬅️ Previous: REST APIs](./02-rest-apis.md) | [⬆️ Index](../../00-INDEX.md) | [Next: gRPC ➡️](./04-grpc.md)
