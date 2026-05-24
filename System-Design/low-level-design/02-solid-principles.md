# SOLID Principles

SOLID is a set of five design principles that make software designs more **understandable**, **flexible**, and **maintainable**. These are the backbone of good Low Level Design.

---

## 1. S — Single Responsibility Principle (SRP)

> **A class should have only one reason to change.**

Each class should do **one thing** and do it well. If a class has multiple responsibilities, changes to one responsibility can break the other.

### Bad Example — Violating SRP

```java
// This class does TOO MANY things
public class Employee {
    private String name;
    private double salary;

    public void calculatePay() {
        // Pay calculation logic
    }

    public void saveToDatabase() {
        // Database persistence logic
    }

    public String generateReport() {
        // Report generation logic
        return "Report for " + name;
    }
}
```

**Problem:** Three reasons to change — pay rules, DB schema, report format.

### Good Example — Following SRP

```java
// Responsibility 1: Employee data
public class Employee {
    private String name;
    private double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() { return name; }
    public double getSalary() { return salary; }
}

// Responsibility 2: Pay calculation
public class PayCalculator {
    public double calculatePay(Employee employee) {
        return employee.getSalary() * 1.0;  // Apply tax, deductions etc.
    }
}

// Responsibility 3: Persistence
public class EmployeeRepository {
    public void save(Employee employee) {
        // Save to database
    }

    public Employee findById(int id) {
        // Fetch from database
        return null;
    }
}

// Responsibility 4: Reporting
public class EmployeeReportGenerator {
    public String generate(Employee employee) {
        return "Report for " + employee.getName();
    }
}
```

### How to Identify SRP Violations
- Class has **many unrelated methods**
- Class name contains **"And"** or **"Manager"** or **"Handler"** (too vague)
- Changing one feature requires modifying unrelated code in the same class
- Class has too many **imports** from different domains

---

## 2. O — Open/Closed Principle (OCP)

> **Software entities should be open for extension, but closed for modification.**

You should be able to **add new behavior** without **changing existing code**. Achieved through abstraction and polymorphism.

### Bad Example — Violating OCP

```java
public class DiscountCalculator {

    public double calculate(String customerType, double amount) {
        if (customerType.equals("REGULAR")) {
            return amount * 0.1;
        } else if (customerType.equals("PREMIUM")) {
            return amount * 0.2;
        } else if (customerType.equals("VIP")) {
            return amount * 0.3;
        }
        // Every new customer type requires modifying this class!
        return 0;
    }
}
```

**Problem:** Adding a new customer type means modifying existing working code.

### Good Example — Following OCP

```java
// Abstraction
public interface DiscountStrategy {
    double calculate(double amount);
}

// Extensions — each in its own class
public class RegularDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.1;
    }
}

public class PremiumDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.2;
    }
}

public class VIPDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.3;
    }
}

// Closed for modification — this class never changes
public class DiscountCalculator {
    private final DiscountStrategy strategy;

    public DiscountCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculate(double amount) {
        return strategy.calculate(amount);
    }
}
```

Now adding a new discount type (e.g., `EmployeeDiscount`) requires only creating a new class — **zero changes** to existing code.

---

## 3. L — Liskov Substitution Principle (LSP)

> **Subtypes must be substitutable for their base types without altering the correctness of the program.**

If class `B` extends class `A`, you should be able to use `B` wherever `A` is expected without breaking anything.

### Bad Example — Violating LSP

```java
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}

public class Ostrich extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Ostrich can't fly!");
    }
}

// Client code breaks
public void makeBirdFly(Bird bird) {
    bird.fly();  // Throws exception if bird is Ostrich!
}
```

**Problem:** `Ostrich` IS-A `Bird` but cannot fly. Substituting `Ostrich` for `Bird` breaks the program.

### Good Example — Following LSP

```java
public abstract class Bird {
    public abstract void eat();
}

public interface Flyable {
    void fly();
}

public class Sparrow extends Bird implements Flyable {
    @Override
    public void eat() {
        System.out.println("Sparrow eating seeds");
    }

    @Override
    public void fly() {
        System.out.println("Sparrow flying");
    }
}

public class Ostrich extends Bird {
    @Override
    public void eat() {
        System.out.println("Ostrich eating plants");
    }
    // No fly() method — Ostrich is not Flyable
}
```

### LSP Rules
- Subclass must **not** throw new exceptions that the parent doesn't throw
- Subclass must **honor** the contract (preconditions, postconditions) of the parent
- Subclass can **weaken** preconditions (accept more) but **not strengthen** them
- Subclass can **strengthen** postconditions (guarantee more) but **not weaken** them

### Classic LSP Violation — Rectangle/Square Problem

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Forces height = width
    }

    @Override
    public void setHeight(int height) {
        this.height = height;
        this.width = height;  // Forces width = height
    }
}

// This breaks with Square
public void testArea(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.getArea() == 20;  // FAILS for Square (gives 16)
}
```

**Fix:** Don't make `Square` extend `Rectangle`. Use a common `Shape` interface instead.

---

## 4. I — Interface Segregation Principle (ISP)

> **No client should be forced to depend on methods it does not use.**

Prefer **many small, specific interfaces** over one large, general-purpose interface.

### Bad Example — Violating ISP (Fat Interface)

```java
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeReport();
}

// Robot is forced to implement methods that don't apply
public class Robot implements Worker {
    @Override
    public void work() { System.out.println("Robot working"); }

    @Override
    public void eat() { /* Not applicable! */ }

    @Override
    public void sleep() { /* Not applicable! */ }

    @Override
    public void attendMeeting() { /* Not applicable! */ }

    @Override
    public void writeReport() { /* Not applicable! */ }
}
```

### Good Example — Following ISP

```java
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface Meetable {
    void attendMeeting();
}

// Human implements all relevant interfaces
public class HumanWorker implements Workable, Eatable, Sleepable, Meetable {
    @Override
    public void work() { System.out.println("Human working"); }

    @Override
    public void eat() { System.out.println("Human eating"); }

    @Override
    public void sleep() { System.out.println("Human sleeping"); }

    @Override
    public void attendMeeting() { System.out.println("Human in meeting"); }
}

// Robot implements only what it needs
public class RobotWorker implements Workable {
    @Override
    public void work() { System.out.println("Robot working"); }
}
```

### How to Identify ISP Violations
- Classes implement interfaces with **empty method bodies**
- Classes throw `UnsupportedOperationException` for interface methods
- Interface has **too many methods** serving different purposes
- Changes to one method in the interface affect unrelated implementing classes

---

## 5. D — Dependency Inversion Principle (DIP)

> **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
> **Abstractions should not depend on details. Details should depend on abstractions.**

### Bad Example — Violating DIP

```java
// Low-level module
public class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

// High-level module directly depends on low-level module
public class OrderService {
    private MySQLDatabase database = new MySQLDatabase();  // Tight coupling!

    public void createOrder(String orderData) {
        // Business logic
        database.save(orderData);  // Directly depends on MySQL
    }
}
```

**Problem:** Switching from MySQL to PostgreSQL requires changing `OrderService`.

### Good Example — Following DIP

```java
// Abstraction (interface)
public interface Database {
    void save(String data);
    String findById(String id);
}

// Low-level module depends on abstraction
public class MySQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }

    @Override
    public String findById(String id) {
        return "MySQL result";
    }
}

public class PostgreSQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to PostgreSQL: " + data);
    }

    @Override
    public String findById(String id) {
        return "PostgreSQL result";
    }
}

// High-level module depends on abstraction
public class OrderService {
    private final Database database;  // Depends on abstraction

    // Dependency injected via constructor
    public OrderService(Database database) {
        this.database = database;
    }

    public void createOrder(String orderData) {
        // Business logic
        database.save(orderData);
    }
}

// Usage — easy to swap implementations
OrderService service1 = new OrderService(new MySQLDatabase());
OrderService service2 = new OrderService(new PostgreSQLDatabase());
```

### Three Ways to Inject Dependencies

```java
// 1. Constructor Injection (Preferred)
public class OrderService {
    private final Database database;
    public OrderService(Database database) {
        this.database = database;
    }
}

// 2. Setter Injection
public class OrderService {
    private Database database;
    public void setDatabase(Database database) {
        this.database = database;
    }
}

// 3. Interface Injection
public interface DatabaseInjector {
    void inject(Database database);
}

public class OrderService implements DatabaseInjector {
    private Database database;
    @Override
    public void inject(Database database) {
        this.database = database;
    }
}
```

---

## SOLID — Quick Reference

| Principle | One-Liner | Key Technique |
|-----------|-----------|---------------|
| **SRP** | One class, one responsibility | Split classes by concern |
| **OCP** | Extend, don't modify | Interfaces + polymorphism |
| **LSP** | Subtypes must be substitutable | Proper hierarchy design |
| **ISP** | Don't force unused methods | Small, focused interfaces |
| **DIP** | Depend on abstractions | Constructor injection |

---

## Common Interview Questions

1. **How do SRP and ISP relate?** — SRP is about classes, ISP is about interfaces. Both aim to keep units focused.
2. **How do OCP and DIP relate?** — DIP enables OCP. By depending on abstractions, you can extend without modifying.
3. **Give a real-world LSP violation** — `java.util.Stack` extends `Vector`, but Stack shouldn't allow random index insertion.
4. **Which principle is most important?** — All are important, but DIP has the biggest architectural impact.
