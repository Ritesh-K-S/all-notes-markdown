# Modular Monolith — Best of Both Worlds

> **What you'll learn**: How to build a single deployable application with well-defined module boundaries, getting the organizational benefits of microservices without the operational complexity of distributed systems.

---

## Real-Life Analogy

Think of an **apartment building**:

- **Big Ball of Mud monolith** = an open-plan warehouse where everyone lives and works in the same space. No walls, no privacy, everything is shared. Chaos.
- **Microservices** = separate houses in different neighborhoods. Each house is independent, but communication requires driving across town (network calls). Expensive, slow coordination.
- **Modular Monolith** = an apartment building. Everyone lives in **one building** (one deployment), but each apartment has **its own walls, kitchen, and bathroom** (modules). Neighbors interact through well-defined channels — doorbell, mailbox — not by walking into each other's kitchens.

You get **clear boundaries** (like microservices) with **simple deployment** (like a monolith).

---

## Core Concept Explained Step-by-Step

### What is a Modular Monolith?

A **modular monolith** is a single deployable application where the code is organized into **strictly separated modules**, each with:
- Its own domain logic
- Its own data (even if in the same database)
- Well-defined public APIs (interfaces)
- No direct access to another module's internals

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SINGLE DEPLOYABLE APPLICATION                     │
│                                                                     │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐    │
│  │   USERS        │  │   ORDERS       │  │    INVENTORY       │    │
│  │   MODULE       │  │   MODULE       │  │    MODULE          │    │
│  │                │  │                │  │                    │    │
│  │ ┌──────────┐  │  │ ┌──────────┐  │  │ ┌──────────────┐  │    │
│  │ │Public API│  │  │ │Public API│  │  │ │ Public API   │  │    │
│  │ │(interface)│  │  │ │(interface)│  │  │ │ (interface)  │  │    │
│  │ └──────────┘  │  │ └──────────┘  │  │ └──────────────┘  │    │
│  │ ┌──────────┐  │  │ ┌──────────┐  │  │ ┌──────────────┐  │    │
│  │ │ Internal │  │  │ │ Internal │  │  │ │   Internal   │  │    │
│  │ │ Services │  │  │ │ Services │  │  │ │   Services   │  │    │
│  │ └──────────┘  │  │ └──────────┘  │  │ └──────────────┘  │    │
│  │ ┌──────────┐  │  │ ┌──────────┐  │  │ ┌──────────────┐  │    │
│  │ │  Data    │  │  │ │  Data    │  │  │ │    Data      │  │    │
│  │ │ (schema) │  │  │ │ (schema) │  │  │ │   (schema)   │  │    │
│  │ └──────────┘  │  │ └──────────┘  │  │ └──────────────┘  │    │
│  └────────────────┘  └────────────────┘  └────────────────────┘    │
│         │                    │                     │                │
│         └────────────────────┼─────────────────────┘                │
│                              │                                      │
│                    ┌─────────▼──────────┐                           │
│                    │  Shared Database   │                           │
│                    │  (but separate     │                           │
│                    │   schemas/tables)  │                           │
│                    └────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────┘
```

### The Spectrum of Architecture

```
                     MODULAR MONOLITH
                          ↓
◀───────────────────────────────────────────────────────────▶
Big Ball     Layered      Modular         SOA       Microservices
of Mud       Monolith     Monolith

Less Structure ───────────────────────────────── More Distribution

Single Process ────────────────────────────── Multiple Processes
Simple Deployment ─────────────────────────── Complex Deployment
```

### Key Rules

| Rule | Description |
|---|---|
| **Module encapsulation** | Each module hides its internal implementation |
| **Public API only** | Modules communicate ONLY through defined interfaces |
| **No cross-module data access** | Module A cannot query Module B's tables directly |
| **Independent development** | Each module can be developed by a separate team |
| **Single deployment** | Everything deploys as one artifact |
| **Enforced boundaries** | Use compile-time checks or architecture tests |

---

## How It Works Internally

### Module Communication Patterns

```
PATTERN 1: Direct Method Call (via interface)
┌──────────────┐     call interface      ┌──────────────┐
│ Order Module │ ─────────────────────▶  │ User Module  │
│              │     UserModule.getUser() │              │
│              │ ◀───────────────────── │              │
│              │     return UserDto       │              │
└──────────────┘                         └──────────────┘

PATTERN 2: In-Process Events (decoupled)
┌──────────────┐     publish event       ┌──────────────────┐
│ Order Module │ ─────────────────────▶  │ Event Bus        │
│              │     "OrderCreated"       │ (in-memory)      │
└──────────────┘                         └────────┬─────────┘
                                                  │
                                    ┌─────────────┼─────────────┐
                                    ▼             ▼             ▼
                             ┌───────────┐ ┌──────────┐ ┌───────────┐
                             │Inventory  │ │Notification│ │ Analytics │
                             │Module     │ │Module     │ │ Module    │
                             └───────────┘ └──────────┘ └───────────┘

PATTERN 3: Shared Kernel (small shared concepts)
┌──────────────┐                         ┌──────────────┐
│ Order Module │          uses           │ User Module  │
│              │ ◀─────────────────────▶ │              │
└──────────────┘    Shared Kernel:       └──────────────┘
                    - UserId (value object)
                    - Money (value object)
                    - Address (value object)
```

### Data Isolation Strategies

```
STRATEGY 1: Separate schemas (same database)
┌─────────────────────────────────────────┐
│           PostgreSQL                     │
│                                         │
│  Schema: users_module                   │
│  ┌─────────────┬──────────────┐         │
│  │   users     │  user_prefs  │         │
│  └─────────────┴──────────────┘         │
│                                         │
│  Schema: orders_module                  │
│  ┌─────────────┬──────────────┐         │
│  │   orders    │ order_items  │         │
│  └─────────────┴──────────────┘         │
│                                         │
│  Schema: inventory_module               │
│  ┌─────────────┬──────────────┐         │
│  │   products  │    stock     │         │
│  └─────────────┴──────────────┘         │
└─────────────────────────────────────────┘

RULE: Order module CANNOT directly query users_module.users!
      It must call UserModule.getUser(id) through the public API.


STRATEGY 2: Table prefix convention
┌─────────────────────────────────────────┐
│  usr_users, usr_preferences             │ ← User module owns these
│  ord_orders, ord_items, ord_payments    │ ← Order module owns these
│  inv_products, inv_stock, inv_warehouses│ ← Inventory module owns these
└─────────────────────────────────────────┘


STRATEGY 3: Enforced via database users/permissions
- db_user "order_svc" can only access order_* tables
- db_user "user_svc" can only access user_* tables
```

### Enforcing Module Boundaries

```
COMPILE-TIME ENFORCEMENT (Java modules / package-private):

com.myapp.modules.users/          ← Java module
├── module-info.java              ← exports ONLY public API
│   exports com.myapp.modules.users.api;
│   // Internal packages NOT exported!
├── api/
│   ├── UserModuleApi.java        ← PUBLIC (other modules use this)
│   └── UserDto.java              ← PUBLIC
└── internal/
    ├── UserService.java          ← PRIVATE (cannot be accessed!)
    ├── UserRepository.java       ← PRIVATE
    └── UserEntity.java           ← PRIVATE

If OrderModule tries to import UserService → COMPILE ERROR! ✓
```

---

## Code Examples

### Python (Modular Monolith)

```python
# === MODULE STRUCTURE ===
# app/
# ├── modules/
# │   ├── users/
# │   │   ├── __init__.py        ← Public API exported here
# │   │   ├── api.py             ← Public interface
# │   │   ├── service.py         ← Internal (not imported by others)
# │   │   ├── repository.py      ← Internal
# │   │   └── models.py          ← Internal
# │   ├── orders/
# │   │   ├── __init__.py
# │   │   ├── api.py
# │   │   ├── service.py
# │   │   └── models.py
# │   └── inventory/
# │       ├── __init__.py
# │       ├── api.py
# │       └── service.py
# └── shared/
#     ├── events.py              ← In-process event bus
#     └── types.py               ← Shared value objects

# === modules/users/api.py (PUBLIC interface) ===
from dataclasses import dataclass

@dataclass
class UserDto:
    id: str
    name: str
    email: str

class UserModuleApi:
    """Public API of the Users module. Other modules use ONLY this."""
    
    def __init__(self, user_service):
        self._service = user_service  # Internal, not exposed
    
    def get_user(self, user_id: str) -> UserDto:
        """Get user info (only what other modules need to know)."""
        user = self._service.find_by_id(user_id)
        return UserDto(id=user.id, name=user.name, email=user.email)
    
    def user_exists(self, user_id: str) -> bool:
        return self._service.exists(user_id)

# === modules/orders/service.py (uses UserModule's PUBLIC API only) ===
from app.modules.users.api import UserModuleApi  # ✅ Import PUBLIC API
# from app.modules.users.repository import UserRepository  # ❌ FORBIDDEN!
from app.shared.events import EventBus

class OrderService:
    def __init__(self, user_api: UserModuleApi, order_repo, event_bus: EventBus):
        self.user_api = user_api       # Depends on PUBLIC interface
        self.order_repo = order_repo   # Own repository
        self.event_bus = event_bus     # In-process event bus
    
    def place_order(self, user_id: str, items: list) -> str:
        # Call User module through its PUBLIC API
        if not self.user_api.user_exists(user_id):
            raise ValueError("User not found")
        
        user = self.user_api.get_user(user_id)
        
        # Order module's own business logic
        order = Order(customer_id=user_id, items=items)
        order.calculate_total()
        self.order_repo.save(order)
        
        # Publish event (other modules react without coupling)
        self.event_bus.publish("OrderCreated", {
            "order_id": order.id,
            "customer_email": user.email,
            "total": order.total
        })
        
        return order.id

# === shared/events.py (in-process event bus) ===
class EventBus:
    """Simple in-process pub/sub for module communication."""
    
    def __init__(self):
        self._handlers: dict[str, list] = {}
    
    def subscribe(self, event_type: str, handler):
        self._handlers.setdefault(event_type, []).append(handler)
    
    def publish(self, event_type: str, data: dict):
        for handler in self._handlers.get(event_type, []):
            handler(data)  # Same process — just a function call!
```

### Java (Spring Boot Modular Monolith)

```java
// === MODULE PUBLIC API ===
// modules/users/api/UserModuleApi.java
public interface UserModuleApi {
    UserDto getUser(String userId);
    boolean userExists(String userId);
}

// modules/users/api/UserDto.java
public record UserDto(String id, String name, String email) {}

// === MODULE INTERNAL IMPLEMENTATION ===
// modules/users/internal/UserModuleImpl.java
@Service
class UserModuleImpl implements UserModuleApi {
    private final UserRepository userRepo;  // Internal!
    
    @Override
    public UserDto getUser(String userId) {
        UserEntity entity = userRepo.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        return new UserDto(entity.getId(), entity.getName(), entity.getEmail());
    }
    
    @Override
    public boolean userExists(String userId) {
        return userRepo.existsById(userId);
    }
}

// === ORDER MODULE USES USER MODULE'S PUBLIC API ===
// modules/orders/internal/OrderService.java
@Service
class OrderService {
    private final UserModuleApi userModule;  // ✅ Public API interface
    private final OrderRepository orderRepo;
    private final ApplicationEventPublisher events;
    
    @Transactional
    public String placeOrder(String userId, List<OrderItem> items) {
        // Call through public API (not directly to UserRepository!)
        if (!userModule.userExists(userId)) {
            throw new IllegalArgumentException("User not found");
        }
        UserDto user = userModule.getUser(userId);
        
        // Own business logic
        Order order = new Order(userId, items);
        order.calculateTotal();
        orderRepo.save(order);
        
        // Publish in-process event
        events.publishEvent(new OrderCreatedEvent(
            order.getId(), user.email(), order.getTotal()
        ));
        
        return order.getId();
    }
}

// === INVENTORY MODULE REACTS TO ORDER EVENTS ===
// modules/inventory/internal/InventoryEventHandler.java
@Component
class InventoryEventHandler {
    private final StockRepository stockRepo;
    
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // React to order without knowing about Order module internals
        for (OrderItemDto item : event.getItems()) {
            stockRepo.decrementStock(item.productId(), item.quantity());
        }
    }
}
```

### Architecture Test (Enforce Boundaries)

```java
// ArchitectureTest.java — Fails if boundaries are violated
@AnalyzeClasses(packages = "com.myapp")
public class ModuleBoundaryTest {
    
    @ArchTest
    static final ArchRule orderModuleDoesNotAccessUserInternals =
        noClasses()
            .that().resideInAPackage("..orders..")
            .should().accessClassesThat()
            .resideInAPackage("..users.internal..");  // ← FORBIDDEN!
    
    @ArchTest
    static final ArchRule modulesOnlyCommunicateThroughApi =
        noClasses()
            .that().resideInAPackage("..internal..")
            .should().beAccessedByClassesThat()
            .resideOutsideOfPackage("..internal..");
    
    @ArchTest
    static final ArchRule noCircularDependencies =
        slices().matching("com.myapp.modules.(*)..")
            .should().beFreeOfCycles();
}
```

---

## Infrastructure Example

### Deployment — Still Simple

```yaml
# docker-compose.yml — One service, one deployment
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
  
  redis:
    image: redis:7
    # Used for caching, NOT for inter-module communication
    # (modules communicate in-process)
```

### Database Migration Per Module

```sql
-- migrations/users/V1__create_users.sql
CREATE SCHEMA IF NOT EXISTS users_module;

CREATE TABLE users_module.users (
    id UUID PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE
);

-- migrations/orders/V1__create_orders.sql
CREATE SCHEMA IF NOT EXISTS orders_module;

CREATE TABLE orders_module.orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,  -- References users, but NO foreign key!
    total DECIMAL(10,2),
    status VARCHAR(20)
);
-- NOTE: No FOREIGN KEY to users_module.users!
-- Data integrity enforced at application level through module API.
```

---

## Real-World Example

### Shopify — 3M Lines of Code, One Monolith

Shopify chose the modular monolith approach for their 3+ million LOC Rails application:

```
SHOPIFY'S MODULAR MONOLITH:

┌──────────────────────────────────────────────────────┐
│              SHOPIFY MONOLITH                         │
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ Checkout │ │ Payments │ │ Shipping │ │ Admin  │ │
│  │ Module   │ │ Module   │ │ Module   │ │ Module │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │ Products │ │ Customers│ │ Marketing│ │ Billing│ │
│  │ Module   │ │ Module   │ │ Module   │ │ Module │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│                                                      │
│  Boundaries enforced by "packwerk" tool              │
│  (Ruby architecture enforcement)                     │
└──────────────────────────────────────────────────────┘
```

- They built **Packwerk** (open-source) to enforce module boundaries in Ruby
- Each module has clear ownership and defined interfaces
- 500+ engineers work on the same codebase without stepping on each other

### Basecamp / HEY

37signals (Basecamp, HEY email) advocates for the modular monolith:
- Single Rails app with clear module boundaries
- Deploys in seconds (one artifact)
- Small team manages a product serving millions

---

## Common Mistakes / Pitfalls

### 1. Modules That Share Everything
❌ **Mistake**: Modules directly query each other's database tables.
✅ **Fix**: Communication ONLY through defined public APIs. No cross-module SQL JOINs.

```
BAD (cross-module data access):
Order Module → SELECT * FROM users_module.users WHERE id = ?

GOOD (through public API):
Order Module → userModule.getUser(userId)
```

### 2. No Boundary Enforcement
❌ **Mistake**: Defining module boundaries in documentation but not enforcing them in code.
✅ **Fix**: Use ArchUnit (Java), Packwerk (Ruby), or module systems to make violations a compile/test error.

### 3. The God Module
❌ **Mistake**: One module grows to contain 80% of the codebase.
✅ **Fix**: If a module is too large, split it. Module size should be manageable by 1-2 teams.

### 4. Circular Dependencies Between Modules
❌ **Mistake**: Users module depends on Orders, Orders depends on Users.
✅ **Fix**: Use events for reverse dependencies, or extract shared concepts into a Shared Kernel.

```
BAD (circular):                    GOOD (event-based):
Users ◀──────▶ Orders              Users ───▶ EventBus ◀─── Orders
                                         publishes    subscribes
```

### 5. Over-Modularizing
❌ **Mistake**: Creating 50 tiny modules for a small application.
✅ **Fix**: Start with 3-7 modules aligned to business domains. Split further only when needed.

---

## When to Use / When NOT to Use

### ✅ Use Modular Monolith When:

| Criteria | Why |
|---|---|
| **Growing codebase (50K-500K LOC)** | Need structure without distribution complexity |
| **10-50 developers** | Teams need boundaries but not full microservice overhead |
| **Not ready for microservices** | Need simpler ops while maintaining code quality |
| **Clear business domains** | Natural module boundaries exist |
| **Planning future microservice migration** | Modules become services later |
| **Value simple deployment** | One deploy, one monitoring stack |

### ❌ Avoid When:

| Criteria | Why |
|---|---|
| **Small app (< 10K LOC)** | Over-engineering; simple layered architecture is fine |
| **Need independent scaling per feature** | All modules scale together in a monolith |
| **Need polyglot tech** | One codebase = one language |
| **Independent deployment required** | All modules deploy together |
| **Very large org (200+ devs)** | Likely need actual microservices |

---

## Modular Monolith → Microservices Migration Path

```
STEP 1: Modular Monolith (today)
┌─────────────────────────────────┐
│  [Users] [Orders] [Inventory]   │  ← One deployment
└─────────────────────────────────┘

STEP 2: Extract one module (when it needs independent scaling)
┌─────────────────────────────────┐    ┌──────────────┐
│  [Users] [Orders]               │────│  Inventory   │
│                                 │    │  Service     │
└─────────────────────────────────┘    └──────────────┘

STEP 3: Extract more as needed
┌─────────────┐  ┌─────────┐  ┌───────────┐  ┌──────────────┐
│   Users     │  │ Orders  │  │ Inventory │  │  Payment     │
│   Service   │  │ Service │  │  Service  │  │  Service     │
└─────────────┘  └─────────┘  └───────────┘  └──────────────┘

Because modules already have clean boundaries,
extraction is straightforward!
```

---

## Key Takeaways

- 🏗️ **Modular monolith = one deployment + strong internal boundaries** — organized into modules that cannot access each other's internals.
- 🚪 **Modules communicate through public APIs** — never through direct database access or internal class imports.
- 🧪 **Boundaries must be ENFORCED** — use architecture tests (ArchUnit), module systems, or tooling (Packwerk) to prevent violations.
- 🗄️ **Each module owns its data** — separate schemas or tables, no cross-module foreign keys.
- 🎯 **It's the sweet spot** for teams of 10-50 developers — clearer than a plain monolith, simpler than microservices.
- 🚀 **Natural stepping stone to microservices** — when you need to extract a service, the boundaries are already defined.
- 🏢 **Shopify, Basecamp, and many others** prove this works at scale with millions of users.

---

## What's Next?

We've built our modular monolith. But what if we already have a messy monolith and want to migrate to microservices gradually? In **Chapter 4.12: Strangler Fig Pattern**, we'll learn the proven approach for incrementally replacing a legacy system without a risky "big bang" rewrite.
