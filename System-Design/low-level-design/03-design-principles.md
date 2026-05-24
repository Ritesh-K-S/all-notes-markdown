# Design Principles & Best Practices

Beyond SOLID, these principles guide everyday design decisions in LLD.

---

## 1. DRY — Don't Repeat Yourself

> **Every piece of knowledge should have a single, unambiguous, authoritative representation in the system.**

Duplication means every change must be made in multiple places — a guaranteed source of bugs.

### Bad Example

```java
public class OrderService {
    public double calculateTotal(List<Item> items) {
        double total = 0;
        for (Item item : items) {
            total += item.getPrice() * item.getQuantity();
        }
        total = total * 1.18;  // 18% tax
        return total;
    }
}

public class InvoiceService {
    public double getInvoiceAmount(List<Item> items) {
        double total = 0;
        for (Item item : items) {
            total += item.getPrice() * item.getQuantity();  // Duplicated!
        }
        total = total * 1.18;  // Duplicated tax logic!
        return total;
    }
}
```

### Good Example

```java
public class PriceCalculator {
    private static final double TAX_RATE = 0.18;

    public double calculateSubtotal(List<Item> items) {
        return items.stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
    }

    public double calculateTotal(List<Item> items) {
        return calculateSubtotal(items) * (1 + TAX_RATE);
    }
}

// Both services use PriceCalculator — single source of truth
```

---

## 2. KISS — Keep It Simple, Stupid

> **The simplest solution that works is the best solution.**

Avoid unnecessary complexity. Don't over-engineer.

### Bad Example — Over-engineered

```java
// Over-engineered for a simple task
public interface StringValidator {
    boolean validate(String input);
}

public class NonEmptyStringValidator implements StringValidator {
    @Override
    public boolean validate(String input) {
        return input != null && !input.trim().isEmpty();
    }
}

public class StringValidatorFactory {
    public static StringValidator create(String type) {
        if ("nonEmpty".equals(type)) {
            return new NonEmptyStringValidator();
        }
        throw new IllegalArgumentException("Unknown type");
    }
}
```

### Good Example — Simple

```java
public class StringUtils {
    public static boolean isNotEmpty(String input) {
        return input != null && !input.trim().isEmpty();
    }
}
```

---

## 3. YAGNI — You Aren't Gonna Need It

> **Don't implement something until you actually need it.**

Don't build features "just in case." It adds complexity, maintenance burden, and is often never used.

### Bad Example

```java
public class User {
    private String name;
    private String email;
    private String phone;
    private String fax;         // Who uses fax anymore?
    private String pager;       // Really?
    private String twitter;     // Not needed yet
    private String instagram;   // Not needed yet
    private String tiktok;      // Not needed yet
    private Map<String, Object> metadata;  // "For future use"
}
```

### Good Example

```java
public class User {
    private String name;
    private String email;
    // Add more fields when the requirement actually comes
}
```

---

## 4. Composition over Inheritance

> **Favor composing objects over extending classes.**

Inheritance creates tight coupling and rigid hierarchies. Composition provides more flexibility.

### Bad Example — Inheritance

```java
public class Logger {
    public void log(String message) {
        System.out.println(message);
    }
}

// Inheriting just to use log()
public class UserService extends Logger {
    public void createUser(String name) {
        log("Creating user: " + name);
        // create user logic
    }
}
// Problem: UserService IS-A Logger? No. UserService USES a Logger.
```

### Good Example — Composition

```java
public class UserService {
    private final Logger logger;  // HAS-A relationship

    public UserService(Logger logger) {
        this.logger = logger;
    }

    public void createUser(String name) {
        logger.log("Creating user: " + name);
        // create user logic
    }
}
```

### Why Composition Wins
- **Flexible** — Can swap implementations at runtime
- **No hierarchy constraints** — A class can compose multiple behaviors
- **Easier to test** — Can mock composed objects
- **Avoids diamond problem** — No multiple inheritance issues

### Composition with Interfaces (Powerful Pattern)

```java
public interface Flyable {
    void fly();
}

public interface Swimmable {
    void swim();
}

public interface Quackable {
    void quack();
}

// Behavior implementations
public class FlyWithWings implements Flyable {
    @Override
    public void fly() { System.out.println("Flying with wings"); }
}

public class NoFly implements Flyable {
    @Override
    public void fly() { System.out.println("Can't fly"); }
}

public class Squeak implements Quackable {
    @Override
    public void quack() { System.out.println("Squeak!"); }
}

// Compose behaviors
public class Duck {
    private Flyable flyBehavior;
    private Quackable quackBehavior;

    public Duck(Flyable flyBehavior, Quackable quackBehavior) {
        this.flyBehavior = flyBehavior;
        this.quackBehavior = quackBehavior;
    }

    public void fly() { flyBehavior.fly(); }
    public void quack() { quackBehavior.quack(); }

    // Can change behavior at runtime!
    public void setFlyBehavior(Flyable flyBehavior) {
        this.flyBehavior = flyBehavior;
    }
}
```

---

## 5. Law of Demeter (Principle of Least Knowledge)

> **A method should only talk to its immediate friends, not strangers.**

An object should only call methods on:
1. Itself
2. Its fields
3. Objects passed as parameters
4. Objects it creates locally

### Bad Example — Train Wreck / Method Chaining

```java
// Reaching deep into object graph
public class OrderProcessor {
    public String getCustomerCity(Order order) {
        return order.getCustomer().getAddress().getCity();  // Violation!
    }
}
```

### Good Example

```java
public class Order {
    private Customer customer;

    public String getCustomerCity() {
        return customer.getCity();  // Delegate to customer
    }
}

public class Customer {
    private Address address;

    public String getCity() {
        return address.getCity();  // Delegate to address
    }
}

public class OrderProcessor {
    public String getCustomerCity(Order order) {
        return order.getCustomerCity();  // Only talks to direct friend
    }
}
```

---

## 6. Separation of Concerns (SoC)

> **Each module/class should address a separate concern.**

Closely related to SRP but at a broader level — applies to layers, modules, and packages.

### Layered Architecture Example

```
┌─────────────────────────────────┐
│   Presentation Layer (Controller)│  ← Handles HTTP requests/responses
├─────────────────────────────────┤
│   Business Logic Layer (Service) │  ← Core business rules
├─────────────────────────────────┤
│   Data Access Layer (Repository) │  ← Database operations
├─────────────────────────────────┤
│   Domain Layer (Entity/Model)    │  ← Data structures
└─────────────────────────────────┘
```

```java
// Each layer has a clear concern

// Controller — handles HTTP
@RestController
public class OrderController {
    private final OrderService orderService;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> create(@RequestBody OrderRequest request) {
        Order order = orderService.createOrder(request);
        return ResponseEntity.ok(order);
    }
}

// Service — handles business logic
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public Order createOrder(OrderRequest request) {
        // Business rules: validate, calculate, process payment
        Order order = new Order(request);
        paymentService.charge(order.getTotal());
        return orderRepository.save(order);
    }
}

// Repository — handles data access
public class OrderRepository {
    public Order save(Order order) {
        // SQL/JPA logic only
        return order;
    }
}
```

---

## 7. Program to an Interface, Not an Implementation

> **Depend on abstractions (interfaces) rather than concrete classes.**

This is the practical application of DIP. It makes code flexible and testable.

```java
// Bad — tied to a specific implementation
ArrayList<String> names = new ArrayList<>();

// Good — depends on abstraction
List<String> names = new ArrayList<>();
// Can easily switch to LinkedList without changing any code that uses 'names'
```

```java
// Bad
public class NotificationService {
    private EmailSender emailSender = new EmailSender();  // Concrete
}

// Good
public class NotificationService {
    private MessageSender messageSender;  // Interface

    public NotificationService(MessageSender messageSender) {
        this.messageSender = messageSender;
    }
}
```

---

## 8. Encapsulate What Varies

> **Identify the aspects of your application that vary and separate them from what stays the same.**

This is the foundation of many design patterns (Strategy, Factory, Observer).

```java
// What varies: tax calculation rules per country
public interface TaxCalculator {
    double calculate(double amount);
}

public class USTaxCalculator implements TaxCalculator {
    @Override
    public double calculate(double amount) { return amount * 0.08; }
}

public class IndiaTaxCalculator implements TaxCalculator {
    @Override
    public double calculate(double amount) { return amount * 0.18; }
}

// What stays the same: order processing flow
public class OrderProcessor {
    private final TaxCalculator taxCalculator;

    public OrderProcessor(TaxCalculator taxCalculator) {
        this.taxCalculator = taxCalculator;
    }

    public double processOrder(double amount) {
        double tax = taxCalculator.calculate(amount);
        return amount + tax;
    }
}
```

---

## Summary Table

| Principle | Core Idea | Anti-Pattern It Prevents |
|-----------|-----------|-------------------------|
| DRY | No duplication | Copy-paste code |
| KISS | Keep it simple | Over-engineering |
| YAGNI | Build only what's needed | Speculative features |
| Composition > Inheritance | Compose, don't extend | Deep hierarchies |
| Law of Demeter | Talk to friends only | Train wreck chains |
| Separation of Concerns | One concern per module | God classes |
| Program to Interface | Depend on abstractions | Tight coupling |
| Encapsulate What Varies | Isolate change | Scattered if-else |
