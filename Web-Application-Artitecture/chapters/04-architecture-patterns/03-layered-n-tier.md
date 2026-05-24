# Layered (N-Tier) Architecture

> **What you'll learn**: How to organize application code into distinct layers (presentation, business, data) so that each layer has a single responsibility and changes in one layer don't ripple through the entire system.

---

## Real-Life Analogy

Think of a **corporate office building**:

- **Ground Floor (Reception)** — Greets visitors, answers phones, routes people to the right department. Doesn't make business decisions.
- **Middle Floors (Business Teams)** — Marketing, sales, HR — they do the actual work, make decisions, apply rules.
- **Basement (Records/Archive)** — Stores all the files, documents, and data. Doesn't care about business rules — just stores and retrieves.

Each floor has a **clear job**. The receptionist doesn't dig through the archive. The archive clerk doesn't make business decisions. Information flows **top-to-bottom** through well-defined channels.

That's **layered architecture** — each layer does one thing, and layers communicate only with the layer directly below them.

---

## Core Concept Explained Step-by-Step

### What is Layered Architecture?

**Layered (N-Tier) architecture** organizes code into horizontal layers, where each layer has a specific responsibility and only depends on the layer below it.

The most common pattern is **3-tier**:

```
┌─────────────────────────────────────────────────────┐
│               PRESENTATION LAYER                    │
│         (Controllers, API Endpoints, Views)         │
│         Handles HTTP requests & responses            │
└───────────────────────┬─────────────────────────────┘
                        │ calls ▼
┌───────────────────────▼─────────────────────────────┐
│                BUSINESS LOGIC LAYER                  │
│          (Services, Domain Models, Rules)            │
│          Contains all application logic              │
└───────────────────────┬─────────────────────────────┘
                        │ calls ▼
┌───────────────────────▼─────────────────────────────┐
│                 DATA ACCESS LAYER                    │
│         (Repositories, DAOs, ORM Queries)           │
│         Talks to databases, APIs, file systems       │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
               ┌─────────────────┐
               │    Database     │
               └─────────────────┘
```

### The Layers Explained

| Layer | Responsibility | Example Classes |
|---|---|---|
| **Presentation** | Accept input, format output, handle HTTP | Controllers, REST endpoints, Serializers |
| **Business Logic** | Rules, validation, workflows, calculations | Services, Domain objects, Validators |
| **Data Access** | CRUD operations, SQL queries, external API calls | Repositories, DAOs, API clients |
| **Database** | Store and retrieve raw data | PostgreSQL, MongoDB, Redis |

### The Key Rule: Dependency Direction

```
ALLOWED:                         FORBIDDEN:
Presentation ──▶ Business        Business ──✗──▶ Presentation
Business     ──▶ Data Access     Data Access ──✗──▶ Business
Data Access  ──▶ Database        Database ──✗──▶ Data Access

Dependencies flow DOWN only. Never UP.
```

This rule ensures that:
- You can **swap the UI** (web → mobile) without touching business logic
- You can **swap the database** (PostgreSQL → MongoDB) without touching business logic
- Business logic is **pure and testable** — no HTTP or SQL mixed in

---

## How a Request Flows Through the Layers

```
HTTP Request: POST /api/users {"name": "Alice", "email": "alice@mail.com"}
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                             │
│                                                                 │
│  UserController.createUser(request)                             │
│    → Parse JSON body                                            │
│    → Validate request format (is email present?)                │
│    → Call business layer                                        │
│    → Format response as JSON                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  BUSINESS LOGIC LAYER                                           │
│                                                                 │
│  UserService.registerUser(name, email)                          │
│    → Check if email already exists (call data layer)            │
│    → Apply business rules (email format, name length)           │
│    → Hash password                                              │
│    → Create User domain object                                  │
│    → Save user (call data layer)                                │
│    → Send welcome email (call notification service)             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  DATA ACCESS LAYER                                              │
│                                                                 │
│  UserRepository.save(user)                                      │
│    → Convert domain object to SQL                               │
│    → Execute: INSERT INTO users (name, email) VALUES (...)      │
│    → Return generated ID                                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   PostgreSQL    │
                    └─────────────────┘
```

---

## How It Works Internally

### Layer Isolation Through Interfaces

Each layer defines **interfaces** (contracts) that the layer above depends on. The implementation can change without affecting callers:

```
┌─────────────────────────────────────────────────────┐
│  BUSINESS LAYER                                     │
│                                                     │
│  class UserService:                                 │
│      def __init__(self, user_repo: UserRepository)  │
│          # Depends on INTERFACE, not implementation │
│                                                     │
└──────────────────────┬──────────────────────────────┘
                       │ depends on interface
                       ▼
         ┌─────────────────────────┐
         │ interface UserRepository │
         │   + save(user): User    │
         │   + find_by_id(id): User│
         │   + find_by_email(): User│
         └─────────────┬───────────┘
                       │ implemented by
            ┌──────────┴──────────┐
            ▼                     ▼
  ┌──────────────────┐  ┌──────────────────┐
  │PostgresUserRepo  │  │  MongoUserRepo   │
  │(SQL queries)     │  │(MongoDB queries) │
  └──────────────────┘  └──────────────────┘
```

### N-Tier Variants

While 3-tier is most common, you can have more layers:

```
4-TIER (with separate API layer):

┌────────────────────────────┐
│  Presentation (UI/Views)   │
├────────────────────────────┤
│  API Layer (Controllers)   │  ← Added: separates UI from API
├────────────────────────────┤
│  Business Logic (Services) │
├────────────────────────────┤
│  Data Access (Repositories)│
└────────────────────────────┘


5-TIER (enterprise):

┌────────────────────────────┐
│  Presentation              │
├────────────────────────────┤
│  API / Controller          │
├────────────────────────────┤
│  Business Logic            │
├────────────────────────────┤
│  Integration / Adapters    │  ← Added: talks to external systems
├────────────────────────────┤
│  Data Access               │
└────────────────────────────┘
```

### Physical vs Logical Tiers

**Logical tiers** = layers in code (packages/folders)
**Physical tiers** = separate machines/processes

```
LOGICAL TIERS (same process):         PHYSICAL TIERS (separate machines):

┌─────────────────────┐              ┌──────────┐    ┌──────────┐    ┌─────┐
│  All layers run     │              │  Web     │───▶│  App     │───▶│ DB  │
│  in ONE process     │              │  Server  │    │  Server  │    │     │
│  (same JAR/binary)  │              │ (Nginx)  │    │ (Tomcat) │    │     │
└─────────────────────┘              └──────────┘    └──────────┘    └─────┘
                                      Machine 1       Machine 2     Machine 3
```

---

## Code Examples

### Python (Flask with Clean Layers)

```python
# === DATA ACCESS LAYER (repositories/user_repository.py) ===
class UserRepository:
    def __init__(self, db_session):
        self.db = db_session
    
    def find_by_email(self, email: str):
        return self.db.query(User).filter(User.email == email).first()
    
    def save(self, user) -> User:
        self.db.add(user)
        self.db.commit()
        return user

# === BUSINESS LOGIC LAYER (services/user_service.py) ===
class UserService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo  # Depends on data layer
    
    def register_user(self, name: str, email: str, password: str) -> dict:
        # Business rule: no duplicate emails
        existing = self.user_repo.find_by_email(email)
        if existing:
            raise ValueError("Email already registered")
        
        # Business rule: password must be 8+ chars
        if len(password) < 8:
            raise ValueError("Password too short")
        
        # Business logic: hash password before storing
        hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
        user = User(name=name, email=email, password_hash=hashed)
        
        saved_user = self.user_repo.save(user)
        return {"id": saved_user.id, "name": saved_user.name}

# === PRESENTATION LAYER (controllers/user_controller.py) ===
@app.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()
    
    # Presentation concern: validate request shape
    if not data.get('email') or not data.get('name'):
        return jsonify({"error": "Missing fields"}), 400
    
    try:
        # Delegate to business layer (NO business logic here!)
        user_service = UserService(UserRepository(db.session))
        result = user_service.register_user(
            data['name'], data['email'], data['password']
        )
        return jsonify(result), 201
    except ValueError as e:
        return jsonify({"error": str(e)}), 400
```

### Java (Spring Boot with Clean Layers)

```java
// === DATA ACCESS LAYER ===
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

// === BUSINESS LOGIC LAYER ===
@Service
public class UserService {
    private final UserRepository userRepo;
    private final PasswordEncoder encoder;
    
    public UserService(UserRepository userRepo, PasswordEncoder encoder) {
        this.userRepo = userRepo;
        this.encoder = encoder;
    }
    
    public UserDto registerUser(String name, String email, String password) {
        // Business rule: no duplicates
        if (userRepo.findByEmail(email).isPresent()) {
            throw new BusinessException("Email already registered");
        }
        // Business rule: password length
        if (password.length() < 8) {
            throw new BusinessException("Password too short");
        }
        // Business logic: hash password
        User user = new User(name, email, encoder.encode(password));
        User saved = userRepo.save(user);
        return new UserDto(saved.getId(), saved.getName(), saved.getEmail());
    }
}

// === PRESENTATION LAYER ===
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @PostMapping
    public ResponseEntity<?> createUser(@RequestBody @Valid CreateUserRequest req) {
        // Only handles HTTP concerns — delegates business logic
        UserDto user = userService.registerUser(
            req.getName(), req.getEmail(), req.getPassword()
        );
        return ResponseEntity.status(201).body(user);
    }
}
```

### Project Structure

```
src/
├── presentation/          ← Layer 1: HTTP concerns only
│   ├── controllers/
│   │   ├── UserController.java
│   │   └── OrderController.java
│   ├── dto/               ← Request/Response shapes
│   │   ├── CreateUserRequest.java
│   │   └── UserResponse.java
│   └── exception/
│       └── GlobalExceptionHandler.java
│
├── business/              ← Layer 2: All business rules
│   ├── services/
│   │   ├── UserService.java
│   │   └── OrderService.java
│   ├── domain/            ← Rich domain objects
│   │   ├── User.java
│   │   └── Order.java
│   └── validators/
│       └── OrderValidator.java
│
└── data/                  ← Layer 3: Database operations
    ├── repositories/
    │   ├── UserRepository.java
    │   └── OrderRepository.java
    ├── entities/          ← DB table mappings
    │   ├── UserEntity.java
    │   └── OrderEntity.java
    └── mappers/
        └── UserMapper.java  ← Entity ↔ Domain conversion
```

---

## Infrastructure Example

### Testing Each Layer Independently

The beauty of layered architecture is **testability**:

```python
# Test BUSINESS layer without database (mock the data layer)
def test_register_user_duplicate_email():
    mock_repo = Mock()
    mock_repo.find_by_email.return_value = User(name="Bob", email="bob@mail.com")
    
    service = UserService(mock_repo)
    
    with pytest.raises(ValueError, match="already registered"):
        service.register_user("Alice", "bob@mail.com", "password123")

# Test DATA layer without business logic (integration test)
def test_save_user_to_database(db_session):
    repo = UserRepository(db_session)
    user = User(name="Alice", email="alice@test.com", password_hash="xxx")
    
    saved = repo.save(user)
    
    assert saved.id is not None
    assert repo.find_by_email("alice@test.com") is not None

# Test PRESENTATION layer without real service (mock business layer)
def test_create_user_endpoint(client):
    with patch('app.UserService.register_user') as mock_svc:
        mock_svc.return_value = {"id": 1, "name": "Alice"}
        
        response = client.post('/api/users', json={
            "name": "Alice", "email": "a@b.com", "password": "12345678"
        })
        
        assert response.status_code == 201
```

---

## Real-World Example

### Java Enterprise Applications

Most **Java enterprise applications** (banks, insurance companies, government systems) follow layered architecture strictly:

```
TYPICAL ENTERPRISE JAVA STACK:

┌─────────────────────────────────────────┐
│  Presentation: Spring MVC / JSF         │
├─────────────────────────────────────────┤
│  Service: EJB / Spring Services         │
├─────────────────────────────────────────┤
│  Data Access: JPA / Hibernate / MyBatis │
├─────────────────────────────────────────┤
│  Database: Oracle / PostgreSQL          │
└─────────────────────────────────────────┘
```

### Django's Built-in Layers

**Django** (Python) follows layered architecture by convention:

| Django Component | Layer |
|---|---|
| `views.py` | Presentation (handles HTTP) |
| `forms.py` / `serializers.py` | Presentation (validates input) |
| `models.py` (business methods) | Business Logic |
| `models.py` (ORM queries) | Data Access |
| `managers.py` | Data Access (custom queries) |

### .NET Enterprise Applications

Microsoft's recommended architecture for ASP.NET follows this exact pattern with **Clean Architecture** layers.

---

## Common Mistakes / Pitfalls

### 1. Business Logic in Controllers
❌ **Mistake**: Putting business rules directly in the presentation layer.

```python
# BAD — business logic leaked into controller
@app.route('/api/orders', methods=['POST'])
def create_order():
    data = request.json
    product = Product.query.get(data['product_id'])
    
    # This is BUSINESS LOGIC — shouldn't be here!
    if product.stock < data['quantity']:
        return jsonify({"error": "Out of stock"}), 400
    if data['quantity'] > 10:
        return jsonify({"error": "Max 10 per order"}), 400
    
    product.stock -= data['quantity']
    # ... 50 more lines of logic in the controller
```

✅ **Fix**: Controller should only call service layer.

### 2. Skipping Layers

❌ **Mistake**: Presentation layer directly accessing the database.

```
WRONG:                              RIGHT:
Controller ──▶ Database             Controller ──▶ Service ──▶ Repository ──▶ DB
(skipped business & data layers)    (proper flow through all layers)
```

### 3. Circular Dependencies

❌ **Mistake**: Business layer importing from presentation layer.
✅ **Fix**: Dependencies ONLY flow downward. Use dependency injection.

### 4. Anemic Domain Model

❌ **Mistake**: Business layer is just "pass-through" with no real logic — services just call repositories.
✅ **Fix**: If your services just do `get from repo → save to repo`, you might be over-engineering. Only use layers when there's actual logic to encapsulate.

### 5. Too Many Layers

❌ **Mistake**: Adding 7+ layers for a simple CRUD app.
✅ **Fix**: 3 layers is sufficient for most applications. Add more only when complexity demands it.

---

## When to Use / When NOT to Use

### ✅ Use Layered Architecture When:

| Criteria | Why |
|---|---|
| **Building a standard web app** | It's the default pattern for a reason |
| **Team has mixed experience** | Easy to understand, well-documented |
| **Clear separation needed** | Different people work on UI vs logic vs DB |
| **Testability is important** | Each layer can be unit tested in isolation |
| **Enterprise applications** | Proven pattern for banks, insurance, government |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Very simple scripts/tools** | Over-engineering for 100-line programs |
| **High-performance real-time systems** | Layer indirection adds latency |
| **Complex domain with many cross-cutting concerns** | Consider Hexagonal Architecture (Chapter 4.9) |
| **Event-driven systems** | Layers assume request-response; events don't fit neatly |
| **Microservices internals** | Each microservice is small enough to not need layers |

---

## Key Takeaways

- 📚 **Layered architecture separates code by responsibility** — presentation, business logic, and data access each get their own layer.
- ⬇️ **Dependencies flow DOWN only** — presentation depends on business, business depends on data. Never upward.
- 🧪 **Each layer is independently testable** — mock the layer below and test in isolation.
- 🔄 **You can swap implementations** — change from PostgreSQL to MongoDB by only modifying the data access layer.
- 📁 **It's the default pattern** for most web applications — Spring Boot, Django, Rails, and ASP.NET all encourage this structure.
- ⚠️ **Don't put business logic in controllers** — the #1 most common violation. Controllers should only handle HTTP concerns.
- 🎯 **3 layers is usually enough** — don't over-engineer with 7 layers unless you have a genuine reason.

---

## What's Next?

Layered architecture works great within a single application. But what happens when you have **multiple applications** that need to work together? In **Chapter 4.4: Service-Oriented Architecture (SOA)**, we'll explore how organizations connect multiple large applications through shared services and an enterprise service bus.
