# Creational Design Patterns

Creational patterns deal with **object creation mechanisms** — creating objects in a manner suitable to the situation, while hiding the creation logic.

---

## 1. Singleton Pattern

> **Ensure a class has only ONE instance and provide a global point of access to it.**

### When to Use
- Database connection pools
- Configuration managers
- Logger instances
- Cache managers
- Thread pools

### Basic Implementation (Not Thread-Safe)

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;

    // Private constructor prevents external instantiation
    private DatabaseConnection() {
        // Initialize connection
    }

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }

    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}
```

**Problem:** Two threads can enter `if (instance == null)` simultaneously and create two instances.

### Thread-Safe — Synchronized Method

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;

    private DatabaseConnection() {}

    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
}
```

**Problem:** `synchronized` on every call is expensive — only needed during creation.

### Thread-Safe — Double-Checked Locking (Best Approach)

```java
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;

    private DatabaseConnection() {}

    public static DatabaseConnection getInstance() {
        if (instance == null) {                     // First check (no lock)
            synchronized (DatabaseConnection.class) {
                if (instance == null) {             // Second check (with lock)
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}
```

**Why `volatile`?** Prevents instruction reordering — ensures the object is fully constructed before the reference is published.

### Thread-Safe — Bill Pugh Singleton (Recommended)

```java
public class DatabaseConnection {

    private DatabaseConnection() {}

    // Inner static class — loaded only when getInstance() is called
    private static class Holder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Why this works:** JVM guarantees that the inner class is loaded only on first access, and class loading is thread-safe.

### Thread-Safe — Enum Singleton (Simplest)

```java
public enum DatabaseConnection {
    INSTANCE;

    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

// Usage
DatabaseConnection.INSTANCE.query("SELECT * FROM users");
```

**Advantages:** Thread-safe, serialization-safe, reflection-proof.

### Singleton Summary

| Approach | Thread-Safe | Lazy Loading | Reflection-Safe |
|----------|------------|-------------|----------------|
| Basic | No | Yes | No |
| Synchronized | Yes | Yes | No |
| Double-Checked | Yes | Yes | No |
| Bill Pugh | Yes | Yes | No |
| Enum | Yes | No | Yes |

---

## 2. Factory Method Pattern

> **Define an interface for creating objects, but let subclasses decide which class to instantiate.**

The Factory Method lets a class defer instantiation to subclasses.

### When to Use
- Object type is determined at runtime
- Hide the instantiation logic from the client
- Multiple classes share a common interface

### Simple Factory (Not a GoF pattern, but commonly used)

```java
// Product interface
public interface Notification {
    void send(String message);
}

// Concrete products
public class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

public class SMSNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

public class PushNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Push: " + message);
    }
}

// Simple Factory
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS" -> new SMSNotification();
            case "PUSH" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage
Notification notification = NotificationFactory.create("EMAIL");
notification.send("Hello!");
```

### Factory Method Pattern (GoF)

```java
// Product
public interface Transport {
    void deliver(String cargo);
}

public class Truck implements Transport {
    @Override
    public void deliver(String cargo) {
        System.out.println("Delivering " + cargo + " by road in a truck");
    }
}

public class Ship implements Transport {
    @Override
    public void deliver(String cargo) {
        System.out.println("Delivering " + cargo + " by sea in a ship");
    }
}

// Creator — abstract class with factory method
public abstract class Logistics {

    // Factory Method — subclasses decide what to create
    public abstract Transport createTransport();

    // Common business logic that uses the factory method
    public void planDelivery(String cargo) {
        Transport transport = createTransport();  // Factory method call
        transport.deliver(cargo);
    }
}

// Concrete Creators
public class RoadLogistics extends Logistics {
    @Override
    public Transport createTransport() {
        return new Truck();
    }
}

public class SeaLogistics extends Logistics {
    @Override
    public Transport createTransport() {
        return new Ship();
    }
}

// Usage
Logistics logistics = new RoadLogistics();
logistics.planDelivery("Electronics");  // Delivers by truck

logistics = new SeaLogistics();
logistics.planDelivery("Coal");  // Delivers by ship
```

---

## 3. Abstract Factory Pattern

> **Provide an interface for creating families of related objects without specifying their concrete classes.**

Factory of Factories — creates related objects that belong together.

### When to Use
- System needs to work with multiple **families of related products**
- You want to ensure that products from the same family are used together
- UI toolkit themes (Dark/Light), cross-platform UI elements

### Example — UI Component Factory

```java
// Abstract Products
public interface Button {
    void render();
    void onClick();
}

public interface TextField {
    void render();
    String getValue();
}

public interface CheckBox {
    void render();
    boolean isChecked();
}

// Concrete Products — Dark Theme Family
public class DarkButton implements Button {
    @Override
    public void render() { System.out.println("Rendering dark button"); }
    @Override
    public void onClick() { System.out.println("Dark button clicked"); }
}

public class DarkTextField implements TextField {
    @Override
    public void render() { System.out.println("Rendering dark text field"); }
    @Override
    public String getValue() { return "dark-value"; }
}

public class DarkCheckBox implements CheckBox {
    @Override
    public void render() { System.out.println("Rendering dark checkbox"); }
    @Override
    public boolean isChecked() { return false; }
}

// Concrete Products — Light Theme Family
public class LightButton implements Button {
    @Override
    public void render() { System.out.println("Rendering light button"); }
    @Override
    public void onClick() { System.out.println("Light button clicked"); }
}

public class LightTextField implements TextField {
    @Override
    public void render() { System.out.println("Rendering light text field"); }
    @Override
    public String getValue() { return "light-value"; }
}

public class LightCheckBox implements CheckBox {
    @Override
    public void render() { System.out.println("Rendering light checkbox"); }
    @Override
    public boolean isChecked() { return false; }
}

// Abstract Factory
public interface UIFactory {
    Button createButton();
    TextField createTextField();
    CheckBox createCheckBox();
}

// Concrete Factories
public class DarkThemeFactory implements UIFactory {
    @Override
    public Button createButton() { return new DarkButton(); }
    @Override
    public TextField createTextField() { return new DarkTextField(); }
    @Override
    public CheckBox createCheckBox() { return new DarkCheckBox(); }
}

public class LightThemeFactory implements UIFactory {
    @Override
    public Button createButton() { return new LightButton(); }
    @Override
    public TextField createTextField() { return new LightTextField(); }
    @Override
    public CheckBox createCheckBox() { return new LightCheckBox(); }
}

// Client — works with any factory
public class Application {
    private final Button button;
    private final TextField textField;
    private final CheckBox checkBox;

    public Application(UIFactory factory) {
        this.button = factory.createButton();
        this.textField = factory.createTextField();
        this.checkBox = factory.createCheckBox();
    }

    public void render() {
        button.render();
        textField.render();
        checkBox.render();
    }
}

// Usage
UIFactory factory = new DarkThemeFactory();
Application app = new Application(factory);
app.render();
```

### Factory Method vs Abstract Factory

| Feature | Factory Method | Abstract Factory |
|---------|---------------|-----------------|
| Creates | One product | Family of products |
| Uses | Inheritance | Composition |
| Complexity | Simple | More complex |
| Example | Create one Notification | Create full UI theme |

---

## 4. Builder Pattern

> **Separate the construction of a complex object from its representation, so the same construction process can create different representations.**

### When to Use
- Object has **many optional parameters**
- Object construction is complex (many steps)
- Want to avoid telescoping constructors
- Immutable objects with many fields

### Problem — Telescoping Constructor

```java
// Anti-pattern: too many constructor variants
public class Pizza {
    public Pizza(String size) { }
    public Pizza(String size, boolean cheese) { }
    public Pizza(String size, boolean cheese, boolean pepperoni) { }
    public Pizza(String size, boolean cheese, boolean pepperoni, boolean mushroom) { }
    // Gets worse with every new topping...
}
```

### Solution — Builder Pattern

```java
public class Pizza {
    // Required parameters
    private final String size;

    // Optional parameters
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushroom;
    private final boolean onion;
    private final boolean bacon;
    private final String crustType;
    private final String sauce;

    // Private constructor — only Builder can create Pizza
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.cheese = builder.cheese;
        this.pepperoni = builder.pepperoni;
        this.mushroom = builder.mushroom;
        this.onion = builder.onion;
        this.bacon = builder.bacon;
        this.crustType = builder.crustType;
        this.sauce = builder.sauce;
    }

    // Getters only — immutable object
    public String getSize() { return size; }
    public boolean hasCheese() { return cheese; }
    // ... other getters

    @Override
    public String toString() {
        return "Pizza [size=" + size + ", cheese=" + cheese +
               ", pepperoni=" + pepperoni + ", crust=" + crustType + "]";
    }

    // Static inner Builder class
    public static class Builder {
        // Required
        private final String size;

        // Optional — with defaults
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean mushroom = false;
        private boolean onion = false;
        private boolean bacon = false;
        private String crustType = "REGULAR";
        private String sauce = "TOMATO";

        public Builder(String size) {
            this.size = size;
        }

        public Builder cheese(boolean value) {
            this.cheese = value;
            return this;    // Return this for chaining
        }

        public Builder pepperoni(boolean value) {
            this.pepperoni = value;
            return this;
        }

        public Builder mushroom(boolean value) {
            this.mushroom = value;
            return this;
        }

        public Builder onion(boolean value) {
            this.onion = value;
            return this;
        }

        public Builder bacon(boolean value) {
            this.bacon = value;
            return this;
        }

        public Builder crustType(String value) {
            this.crustType = value;
            return this;
        }

        public Builder sauce(String value) {
            this.sauce = value;
            return this;
        }

        // Build method — creates the final immutable object
        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// Usage — clean, readable, fluent API
Pizza pizza = new Pizza.Builder("LARGE")
        .cheese(true)
        .pepperoni(true)
        .mushroom(true)
        .crustType("THIN")
        .sauce("BBQ")
        .build();

System.out.println(pizza);
```

### Builder with Validation

```java
public Pizza build() {
    // Validate before building
    if (size == null || size.isEmpty()) {
        throw new IllegalStateException("Size is required");
    }
    if (crustType == null) {
        throw new IllegalStateException("Crust type is required");
    }
    return new Pizza(this);
}
```

### Builder in Real Java Code
- `StringBuilder` — builds strings
- `Stream.builder()` — builds streams
- `HttpRequest.newBuilder()` — builds HTTP requests
- Lombok `@Builder` annotation — generates builder automatically

---

## 5. Prototype Pattern

> **Create new objects by copying (cloning) an existing object, instead of creating from scratch.**

### When to Use
- Object creation is **expensive** (DB calls, network calls, complex computation)
- You need copies of objects with **slight variations**
- Registry of pre-configured prototypes

### Implementation Using Cloneable

```java
public abstract class Shape implements Cloneable {
    private String color;
    private int x;
    private int y;

    public Shape() {}

    // Copy constructor
    public Shape(Shape source) {
        this.color = source.color;
        this.x = source.x;
        this.y = source.y;
    }

    @Override
    public abstract Shape clone();

    // Getters and setters
    public String getColor() { return color; }
    public void setColor(String color) { this.color = color; }
    public int getX() { return x; }
    public void setX(int x) { this.x = x; }
    public int getY() { return y; }
    public void setY(int y) { this.y = y; }
}

public class Circle extends Shape {
    private int radius;

    public Circle() {}

    public Circle(Circle source) {
        super(source);
        this.radius = source.radius;
    }

    @Override
    public Circle clone() {
        return new Circle(this);
    }

    public int getRadius() { return radius; }
    public void setRadius(int radius) { this.radius = radius; }
}

public class Rectangle extends Shape {
    private int width;
    private int height;

    public Rectangle() {}

    public Rectangle(Rectangle source) {
        super(source);
        this.width = source.width;
        this.height = source.height;
    }

    @Override
    public Rectangle clone() {
        return new Rectangle(this);
    }

    public int getWidth() { return width; }
    public void setWidth(int width) { this.width = width; }
    public int getHeight() { return height; }
    public void setHeight(int height) { this.height = height; }
}
```

### Prototype Registry

```java
public class ShapeRegistry {
    private final Map<String, Shape> shapes = new HashMap<>();

    public void registerShape(String key, Shape shape) {
        shapes.put(key, shape);
    }

    public Shape getShape(String key) {
        Shape shape = shapes.get(key);
        if (shape == null) {
            throw new IllegalArgumentException("Shape not found: " + key);
        }
        return shape.clone();  // Return a clone, not the original
    }
}

// Usage
ShapeRegistry registry = new ShapeRegistry();

// Register prototypes
Circle redCircle = new Circle();
redCircle.setColor("Red");
redCircle.setRadius(10);
registry.registerShape("RED_CIRCLE", redCircle);

Rectangle blueRect = new Rectangle();
blueRect.setColor("Blue");
blueRect.setWidth(20);
blueRect.setHeight(30);
registry.registerShape("BLUE_RECT", blueRect);

// Clone from registry
Shape circle1 = registry.getShape("RED_CIRCLE");
Shape circle2 = registry.getShape("RED_CIRCLE");

System.out.println(circle1 == circle2);  // false — different objects
```

### Shallow Copy vs Deep Copy

```java
// Shallow Copy — nested objects share reference
public class ShallowCopy {
    public Employee clone(Employee original) {
        Employee copy = new Employee();
        copy.setName(original.getName());
        copy.setAddress(original.getAddress());  // Same reference!
        return copy;
    }
}

// Deep Copy — nested objects are also cloned
public class DeepCopy {
    public Employee clone(Employee original) {
        Employee copy = new Employee();
        copy.setName(original.getName());
        copy.setAddress(new Address(original.getAddress()));  // New object!
        return copy;
    }
}
```

---

## Creational Patterns — Comparison

| Pattern | Purpose | Key Feature |
|---------|---------|-------------|
| **Singleton** | One instance only | Global access point |
| **Factory Method** | Delegate creation to subclass | Polymorphic creation |
| **Abstract Factory** | Create product families | Related objects together |
| **Builder** | Step-by-step complex construction | Fluent API, immutability |
| **Prototype** | Clone existing objects | Avoid expensive creation |

### When to Use Which?

| Scenario | Pattern |
|----------|---------|
| Only one instance needed (DB pool, config) | Singleton |
| Object type decided at runtime | Factory Method |
| Multiple related objects created together | Abstract Factory |
| Object has many optional parameters | Builder |
| Creating object is expensive; need copies | Prototype |
