# OOP Fundamentals

Object-Oriented Programming is the foundation of Low Level Design. It models real-world entities as objects with state (fields) and behavior (methods).

---

## Four Pillars of OOP

---

## 1. Encapsulation

Encapsulation is the bundling of data (fields) and methods that operate on that data into a single unit (class), while **restricting direct access** to the internal state.

### Why Encapsulation?
- **Data Protection** — Prevents external code from putting an object into an invalid state
- **Controlled Access** — Only expose what is necessary via getters/setters
- **Flexibility** — Internal implementation can change without affecting external code
- **Maintainability** — Reduces coupling between components

### Rules
- Make fields `private`
- Provide `public` getter/setter methods only where needed
- Add validation logic inside setters

### Example

```java
public class BankAccount {
    private String accountNumber;
    private double balance;

    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.balance = initialBalance;
    }

    // Only getter — account number should not change
    public String getAccountNumber() {
        return accountNumber;
    }

    public double getBalance() {
        return balance;
    }

    // Controlled modification — no direct setter for balance
    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount > balance) {
            throw new IllegalArgumentException("Insufficient balance");
        }
        this.balance -= amount;
    }
}
```

### Key Points
- `balance` is never set directly — only modified through `deposit()` and `withdraw()`
- Validation ensures the object is always in a valid state
- No setter for `accountNumber` — it's immutable after creation

---

## 2. Abstraction

Abstraction means **hiding complex implementation details** and exposing only the essential features. The user interacts with a simplified interface without knowing the internal workings.

### Why Abstraction?
- **Simplicity** — Users don't need to know how something works internally
- **Reduces Complexity** — Focus on "what" not "how"
- **Loose Coupling** — Code depends on abstractions, not concrete implementations

### Achieved Through
- **Abstract Classes** — Partial abstraction (can have both abstract and concrete methods)
- **Interfaces** — Full abstraction (only method signatures, no implementation)

### Example — Abstract Class

```java
public abstract class PaymentProcessor {
    
    // Template — common flow
    public final void processPayment(double amount) {
        validate(amount);
        deductAmount(amount);
        sendConfirmation();
    }

    private void validate(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
    }

    // Abstract — subclass decides how to deduct
    protected abstract void deductAmount(double amount);

    // Abstract — subclass decides how to confirm
    protected abstract void sendConfirmation();
}
```

### Example — Interface

```java
public interface NotificationService {
    void send(String recipient, String message);
    boolean isDelivered(String messageId);
}

public class EmailNotificationService implements NotificationService {
    @Override
    public void send(String recipient, String message) {
        // SMTP logic hidden from the caller
        System.out.println("Email sent to " + recipient);
    }

    @Override
    public boolean isDelivered(String messageId) {
        // Check delivery status
        return true;
    }
}

public class SMSNotificationService implements NotificationService {
    @Override
    public void send(String recipient, String message) {
        // SMS gateway logic hidden from the caller
        System.out.println("SMS sent to " + recipient);
    }

    @Override
    public boolean isDelivered(String messageId) {
        return true;
    }
}
```

### When to Use Abstract Class vs Interface

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Multiple Inheritance | No | Yes |
| Constructor | Yes | No |
| Fields | Instance variables | Only `static final` |
| Method Implementation | Can have concrete methods | Default methods (Java 8+) |
| Use When | Shared state + behavior among related classes | Defining a contract for unrelated classes |

---

## 3. Inheritance

Inheritance allows a class (child/subclass) to **inherit fields and methods** from another class (parent/superclass). It models the **"is-a"** relationship.

### Why Inheritance?
- **Code Reuse** — Common logic lives in the parent
- **Hierarchical Classification** — Models natural relationships
- **Polymorphism** — Enables treating subclass objects as parent type

### Types of Inheritance in Java
- **Single** — One child, one parent
- **Multilevel** — A → B → C chain
- **Hierarchical** — One parent, multiple children
- **Multiple** — NOT supported via classes (supported via interfaces)

### Example

```java
public class Vehicle {
    private String brand;
    private int year;

    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year = year;
    }

    public void start() {
        System.out.println(brand + " is starting...");
    }

    public String getBrand() { return brand; }
    public int getYear() { return year; }
}

public class Car extends Vehicle {
    private int numDoors;

    public Car(String brand, int year, int numDoors) {
        super(brand, year);  // Call parent constructor
        this.numDoors = numDoors;
    }

    public void openTrunk() {
        System.out.println("Trunk opened");
    }
}

public class ElectricCar extends Car {
    private int batteryCapacity;

    public ElectricCar(String brand, int year, int numDoors, int batteryCapacity) {
        super(brand, year, numDoors);
        this.batteryCapacity = batteryCapacity;
    }

    @Override
    public void start() {
        System.out.println(getBrand() + " starting silently (electric)...");
    }
}
```

### Pitfalls of Inheritance
- **Tight Coupling** — Child is tightly coupled to parent's implementation
- **Fragile Base Class** — Changes in parent can break children
- **Deep Hierarchies** — Hard to understand and maintain
- **Favor Composition over Inheritance** when possible (discussed later)

---

## 4. Polymorphism

Polymorphism means **"many forms"** — the ability of an object to take different forms and behave differently based on context.

### Two Types

### 4a. Compile-Time Polymorphism (Method Overloading)

Same method name, different parameter lists. Resolved at **compile time**.

```java
public class Calculator {
    
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

**Rules for Overloading:**
- Must differ in parameter count, type, or order
- Return type alone is NOT enough to overload
- Can have different access modifiers

### 4b. Runtime Polymorphism (Method Overriding)

Subclass provides its own implementation of a method defined in the parent. Resolved at **runtime** via dynamic dispatch.

```java
public abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    public abstract double area();

    public abstract double perimeter();

    @Override
    public String toString() {
        return this.getClass().getSimpleName() + 
               " [color=" + color + ", area=" + area() + "]";
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
}

public class Rectangle extends Shape {
    private double length;
    private double width;

    public Rectangle(String color, double length, double width) {
        super(color);
        this.length = length;
        this.width = width;
    }

    @Override
    public double area() {
        return length * width;
    }

    @Override
    public double perimeter() {
        return 2 * (length + width);
    }
}
```

### Polymorphism in Action

```java
public class Main {
    public static void main(String[] args) {
        // Parent reference, child object — this is polymorphism
        Shape s1 = new Circle("Red", 5.0);
        Shape s2 = new Rectangle("Blue", 4.0, 6.0);

        // Same method call, different behavior
        System.out.println(s1.area());  // Circle's area
        System.out.println(s2.area());  // Rectangle's area

        // Works with collections too
        List<Shape> shapes = List.of(s1, s2);
        for (Shape s : shapes) {
            System.out.println(s);  // Calls each shape's toString()
        }
    }
}
```

**Rules for Overriding:**
- Same method signature (name + parameters)
- Return type must be same or covariant (subclass)
- Access modifier cannot be more restrictive
- Cannot override `static`, `final`, or `private` methods
- Use `@Override` annotation always

---

## Association, Aggregation & Composition

These define **relationships between objects** — critical in LLD class design.

### Association
A general relationship where one object **uses** or **interacts with** another. No ownership implied.

```java
// Teacher and Student — both can exist independently
public class Teacher {
    private String name;
    
    public void teach(Student student) {
        System.out.println(name + " is teaching " + student.getName());
    }
}
```

### Aggregation (Has-A — Weak Ownership)
A special form of association where one object **contains** another, but the contained object can **exist independently**.

```java
// Department has Professors, but Professors exist independently
public class Department {
    private String name;
    private List<Professor> professors;  // Aggregation

    public Department(String name, List<Professor> professors) {
        this.name = name;
        this.professors = professors;  // Professors created outside
    }
}
```

### Composition (Has-A — Strong Ownership)
A stronger form of aggregation where the contained object **cannot exist** without the container. The container **manages** the lifecycle.

```java
// House has Rooms — Rooms cannot exist without a House
public class House {
    private List<Room> rooms;

    public House(int numberOfRooms) {
        rooms = new ArrayList<>();
        for (int i = 0; i < numberOfRooms; i++) {
            rooms.add(new Room());  // Rooms created inside — House owns them
        }
    }
    // When House is destroyed, Rooms are destroyed too
}
```

### Summary Table

| Relationship | Ownership | Lifecycle | Example |
|-------------|-----------|-----------|---------|
| Association | None | Independent | Teacher ↔ Student |
| Aggregation | Weak | Independent | Department → Professor |
| Composition | Strong | Dependent | House → Room |

---

## Key Takeaways

1. **Encapsulation** = Hide data, expose behavior
2. **Abstraction** = Hide complexity, expose interface
3. **Inheritance** = Reuse via "is-a" hierarchy
4. **Polymorphism** = Same interface, different behavior
5. **Prefer Composition over Inheritance** for flexibility
6. Always think about **relationships** (association, aggregation, composition) when designing classes
