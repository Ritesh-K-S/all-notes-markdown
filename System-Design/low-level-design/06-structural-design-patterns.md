# Structural Design Patterns

Structural patterns deal with **how classes and objects are composed** to form larger structures. They ensure that when parts of a system change, the entire structure doesn't need to change.

---

## 1. Adapter Pattern

> **Convert the interface of a class into another interface that clients expect. Adapter lets incompatible interfaces work together.**

Also called: **Wrapper**

### When to Use
- Integrating a **third-party library** with an incompatible interface
- Making **legacy code** work with new code
- Two existing interfaces don't match but need to work together

### Example — Payment Gateway Integration

```java
// Our application's expected interface
public interface PaymentProcessor {
    void processPayment(double amount, String currency);
    PaymentStatus getStatus(String transactionId);
}

// Third-party payment SDK — incompatible interface
public class StripeSDK {
    public void makeCharge(int amountInCents, String curr, String idempotencyKey) {
        System.out.println("Stripe: Charged " + amountInCents + " cents in " + curr);
    }

    public String checkChargeStatus(String chargeId) {
        return "SUCCEEDED";
    }
}

// Adapter — bridges the gap
public class StripeAdapter implements PaymentProcessor {
    private final StripeSDK stripeSDK;

    public StripeAdapter(StripeSDK stripeSDK) {
        this.stripeSDK = stripeSDK;
    }

    @Override
    public void processPayment(double amount, String currency) {
        // Convert dollars to cents
        int amountInCents = (int) (amount * 100);
        String idempotencyKey = UUID.randomUUID().toString();
        stripeSDK.makeCharge(amountInCents, currency, idempotencyKey);
    }

    @Override
    public PaymentStatus getStatus(String transactionId) {
        String status = stripeSDK.checkChargeStatus(transactionId);
        return PaymentStatus.valueOf(status);
    }
}

// Client code — works with our interface
public class OrderService {
    private final PaymentProcessor paymentProcessor;

    public OrderService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    public void checkout(double amount) {
        paymentProcessor.processPayment(amount, "USD");
    }
}

// Usage — inject the adapter
StripeSDK stripe = new StripeSDK();
PaymentProcessor processor = new StripeAdapter(stripe);
OrderService orderService = new OrderService(processor);
orderService.checkout(99.99);
```

### Two Types of Adapter

| Type | Approach | Java Support |
|------|----------|-------------|
| **Object Adapter** | Uses composition (holds reference) | Preferred |
| **Class Adapter** | Uses inheritance (extends adaptee) | Limited (no multiple inheritance) |

### Real-World Examples
- `Arrays.asList()` — adapts array to List interface
- `InputStreamReader` — adapts InputStream (bytes) to Reader (characters)
- `Collections.enumeration()` — adapts Iterator to Enumeration

---

## 2. Bridge Pattern

> **Decouple an abstraction from its implementation so that the two can vary independently.**

Splits a class into two separate hierarchies: **abstraction** and **implementation**.

### When to Use
- A class has **two dimensions that can change independently**
- You want to avoid an explosion of subclasses
- Switch implementations at runtime

### Problem — Class Explosion Without Bridge

```
Without Bridge (2 shapes × 3 colors = 6 classes):
  RedCircle, BlueCircle, GreenCircle
  RedSquare, BlueSquare, GreenSquare
  
Adding 1 more shape → 3 more classes
Adding 1 more color → 2 more classes
```

### Solution — Bridge Pattern

```java
// Implementation hierarchy
public interface Color {
    String fill();
}

public class RedColor implements Color {
    @Override
    public String fill() { return "Red"; }
}

public class BlueColor implements Color {
    @Override
    public String fill() { return "Blue"; }
}

public class GreenColor implements Color {
    @Override
    public String fill() { return "Green"; }
}

// Abstraction hierarchy
public abstract class Shape {
    protected Color color;  // Bridge to implementation

    public Shape(Color color) {
        this.color = color;
    }

    public abstract void draw();
}

public class Circle extends Shape {
    private double radius;

    public Circle(Color color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public void draw() {
        System.out.println("Drawing " + color.fill() + " circle with radius " + radius);
    }
}

public class Square extends Shape {
    private double side;

    public Square(Color color, double side) {
        super(color);
        this.side = side;
    }

    @Override
    public void draw() {
        System.out.println("Drawing " + color.fill() + " square with side " + side);
    }
}

// Usage — combine any shape with any color
Shape redCircle = new Circle(new RedColor(), 5.0);
Shape blueSquare = new Square(new BlueColor(), 10.0);

redCircle.draw();   // Drawing Red circle with radius 5.0
blueSquare.draw();  // Drawing Blue square with side 10.0
```

### Practical Example — Notification System

```java
// Implementation: HOW to send
public interface MessageSender {
    void sendMessage(String message, String recipient);
}

public class EmailSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}

public class SMSSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

// Abstraction: WHAT type of notification
public abstract class Notification {
    protected MessageSender sender;

    public Notification(MessageSender sender) {
        this.sender = sender;
    }

    public abstract void notify(String recipient);
}

public class UrgentNotification extends Notification {
    private String message;

    public UrgentNotification(MessageSender sender, String message) {
        super(sender);
        this.message = message;
    }

    @Override
    public void notify(String recipient) {
        sender.sendMessage("[URGENT] " + message, recipient);
    }
}

public class RegularNotification extends Notification {
    private String message;

    public RegularNotification(MessageSender sender, String message) {
        super(sender);
        this.message = message;
    }

    @Override
    public void notify(String recipient) {
        sender.sendMessage(message, recipient);
    }
}

// Mix and match freely
Notification urgentEmail = new UrgentNotification(new EmailSender(), "Server is down!");
Notification regularSMS = new RegularNotification(new SMSSender(), "Monthly report ready");
```

---

## 3. Composite Pattern

> **Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.**

### When to Use
- Data has a **tree/hierarchical structure**
- You want to treat **leaf** and **composite** objects the same way
- File systems, organization charts, UI component trees, menu systems

### Example — File System

```java
// Component — common interface for both files and directories
public abstract class FileSystemItem {
    protected String name;

    public FileSystemItem(String name) {
        this.name = name;
    }

    public String getName() { return name; }

    public abstract long getSize();
    public abstract void display(String indent);

    // Default implementations for leaf — overridden by composite
    public void add(FileSystemItem item) {
        throw new UnsupportedOperationException("Cannot add to a file");
    }

    public void remove(FileSystemItem item) {
        throw new UnsupportedOperationException("Cannot remove from a file");
    }
}

// Leaf
public class File extends FileSystemItem {
    private long size;

    public File(String name, long size) {
        super(name);
        this.size = size;
    }

    @Override
    public long getSize() {
        return size;
    }

    @Override
    public void display(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// Composite
public class Directory extends FileSystemItem {
    private List<FileSystemItem> children = new ArrayList<>();

    public Directory(String name) {
        super(name);
    }

    @Override
    public void add(FileSystemItem item) {
        children.add(item);
    }

    @Override
    public void remove(FileSystemItem item) {
        children.remove(item);
    }

    @Override
    public long getSize() {
        // Recursively calculates total size
        return children.stream()
                .mapToLong(FileSystemItem::getSize)
                .sum();
    }

    @Override
    public void display(String indent) {
        System.out.println(indent + "📁 " + name + " (" + getSize() + " bytes)");
        for (FileSystemItem child : children) {
            child.display(indent + "  ");
        }
    }
}

// Usage
Directory root = new Directory("root");
Directory src = new Directory("src");
Directory docs = new Directory("docs");

src.add(new File("Main.java", 1500));
src.add(new File("Utils.java", 800));
docs.add(new File("README.md", 200));

root.add(src);
root.add(docs);
root.add(new File(".gitignore", 50));

root.display("");  // Displays full tree with sizes
```

Output:
```
📁 root (2550 bytes)
  📁 src (2300 bytes)
    📄 Main.java (1500 bytes)
    📄 Utils.java (800 bytes)
  📁 docs (200 bytes)
    📄 README.md (200 bytes)
  📄 .gitignore (50 bytes)
```

---

## 4. Decorator Pattern

> **Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.**

Also called: **Wrapper**

### When to Use
- Add behavior to objects **at runtime** without modifying them
- Avoid **subclass explosion** for combining features
- Extend functionality in a **stackable** way

### Example — Coffee Ordering System

```java
// Component
public interface Coffee {
    String getDescription();
    double getCost();
}

// Concrete Component
public class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() { return "Simple Coffee"; }

    @Override
    public double getCost() { return 50.0; }
}

// Base Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost();
    }
}

// Concrete Decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + " + Milk";
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 15.0;
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + " + Sugar";
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 5.0;
    }
}

public class WhipCreamDecorator extends CoffeeDecorator {
    public WhipCreamDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + " + Whip Cream";
    }

    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 25.0;
    }
}

// Usage — stack decorators dynamically
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " = ₹" + coffee.getCost());
// Simple Coffee = ₹50.0

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " = ₹" + coffee.getCost());
// Simple Coffee + Milk = ₹65.0

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " = ₹" + coffee.getCost());
// Simple Coffee + Milk + Sugar = ₹70.0

coffee = new WhipCreamDecorator(coffee);
System.out.println(coffee.getDescription() + " = ₹" + coffee.getCost());
// Simple Coffee + Milk + Sugar + Whip Cream = ₹95.0
```

### Real-World Examples
- `BufferedInputStream(new FileInputStream(file))` — adds buffering
- `Collections.synchronizedList(list)` — adds thread safety
- `Collections.unmodifiableList(list)` — adds immutability

---

## 5. Facade Pattern

> **Provide a unified, simplified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.**

### When to Use
- Simplify interaction with a **complex subsystem**
- Provide a **clean API** for clients
- Decouple clients from subsystem internals

### Example — Home Theater System

```java
// Complex subsystem classes
public class DVDPlayer {
    public void on() { System.out.println("DVD Player ON"); }
    public void play(String movie) { System.out.println("Playing: " + movie); }
    public void stop() { System.out.println("DVD Player stopped"); }
    public void off() { System.out.println("DVD Player OFF"); }
}

public class Projector {
    public void on() { System.out.println("Projector ON"); }
    public void setInput(String input) { System.out.println("Projector input: " + input); }
    public void off() { System.out.println("Projector OFF"); }
}

public class SoundSystem {
    public void on() { System.out.println("Sound System ON"); }
    public void setVolume(int level) { System.out.println("Volume: " + level); }
    public void off() { System.out.println("Sound System OFF"); }
}

public class Lights {
    public void dim(int level) { System.out.println("Lights dimmed to " + level + "%"); }
    public void on() { System.out.println("Lights ON"); }
}

// Facade — simplifies complex subsystem interaction
public class HomeTheaterFacade {
    private final DVDPlayer dvdPlayer;
    private final Projector projector;
    private final SoundSystem soundSystem;
    private final Lights lights;

    public HomeTheaterFacade(DVDPlayer dvdPlayer, Projector projector,
                            SoundSystem soundSystem, Lights lights) {
        this.dvdPlayer = dvdPlayer;
        this.projector = projector;
        this.soundSystem = soundSystem;
        this.lights = lights;
    }

    // Simple method hides complex sequence
    public void watchMovie(String movie) {
        System.out.println("--- Setting up movie ---");
        lights.dim(10);
        projector.on();
        projector.setInput("DVD");
        soundSystem.on();
        soundSystem.setVolume(50);
        dvdPlayer.on();
        dvdPlayer.play(movie);
    }

    public void endMovie() {
        System.out.println("--- Shutting down ---");
        dvdPlayer.stop();
        dvdPlayer.off();
        soundSystem.off();
        projector.off();
        lights.on();
    }
}

// Usage — client uses ONE simple method
HomeTheaterFacade theater = new HomeTheaterFacade(
    new DVDPlayer(), new Projector(), new SoundSystem(), new Lights()
);

theater.watchMovie("Inception");  // One call does everything
theater.endMovie();
```

---

## 6. Flyweight Pattern

> **Use sharing to support large numbers of fine-grained objects efficiently by sharing common state.**

Splits object state into:
- **Intrinsic state** — shared, immutable, stored in the flyweight
- **Extrinsic state** — unique per context, passed by the client

### When to Use
- Application uses a **large number** of similar objects
- Objects have **shared state** that can be extracted
- Memory optimization is important

### Example — Text Editor Characters

```java
// Flyweight — shared intrinsic state
public class CharacterStyle {
    private final String font;
    private final int size;
    private final String color;

    public CharacterStyle(String font, int size, String color) {
        this.font = font;
        this.size = size;
        this.color = color;
    }

    public void render(char character, int row, int col) {
        System.out.println("Rendering '" + character + "' at (" + row + "," + col + 
                           ") with font=" + font + ", size=" + size + ", color=" + color);
    }

    // equals and hashCode for proper caching
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        CharacterStyle that = (CharacterStyle) o;
        return size == that.size && 
               Objects.equals(font, that.font) && 
               Objects.equals(color, that.color);
    }

    @Override
    public int hashCode() {
        return Objects.hash(font, size, color);
    }
}

// Flyweight Factory — manages shared instances
public class CharacterStyleFactory {
    private static final Map<String, CharacterStyle> cache = new HashMap<>();

    public static CharacterStyle getStyle(String font, int size, String color) {
        String key = font + "-" + size + "-" + color;
        return cache.computeIfAbsent(key, k -> new CharacterStyle(font, size, color));
    }

    public static int getCacheSize() {
        return cache.size();
    }
}

// Usage
// 10,000 characters but only a few unique styles
CharacterStyle style1 = CharacterStyleFactory.getStyle("Arial", 12, "Black");
CharacterStyle style2 = CharacterStyleFactory.getStyle("Arial", 12, "Black");

System.out.println(style1 == style2);  // true — same object (shared)

// Render with extrinsic state (position)
style1.render('H', 0, 0);  // Extrinsic: character and position
style1.render('e', 0, 1);
style1.render('l', 0, 2);

System.out.println("Unique styles in cache: " + CharacterStyleFactory.getCacheSize());
```

### Real-World Examples
- `String.intern()` — shares string instances
- `Integer.valueOf()` — caches -128 to 127
- `Boolean.valueOf()` — returns cached TRUE/FALSE

---

## 7. Proxy Pattern

> **Provide a surrogate or placeholder for another object to control access to it.**

### Types of Proxy

| Type | Purpose |
|------|---------|
| **Virtual Proxy** | Lazy initialization — delays expensive creation |
| **Protection Proxy** | Access control — checks permissions |
| **Remote Proxy** | Represents remote object (RMI, REST client) |
| **Logging Proxy** | Logs method calls before forwarding |
| **Caching Proxy** | Caches results to avoid repeated work |

### Example — Virtual Proxy (Lazy Loading)

```java
// Subject interface
public interface Image {
    void display();
}

// Real Subject — expensive to create
public class HighResolutionImage implements Image {
    private final String filename;
    private byte[] imageData;

    public HighResolutionImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // Expensive operation
    }

    private void loadFromDisk() {
        System.out.println("Loading high-res image: " + filename);
        // Simulate expensive loading
        this.imageData = new byte[1024 * 1024];  // 1MB
    }

    @Override
    public void display() {
        System.out.println("Displaying: " + filename);
    }
}

// Proxy — defers loading until needed
public class ImageProxy implements Image {
    private final String filename;
    private HighResolutionImage realImage;  // Created lazily

    public ImageProxy(String filename) {
        this.filename = filename;
        // NOT loading the image yet
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new HighResolutionImage(filename);  // Load on first use
        }
        realImage.display();
    }
}

// Usage
Image image = new ImageProxy("vacation.jpg");  // No loading happens
// ... much later ...
image.display();  // NOW it loads and displays
```

### Example — Protection Proxy (Access Control)

```java
public interface DocumentService {
    String readDocument(String docId);
    void writeDocument(String docId, String content);
    void deleteDocument(String docId);
}

public class RealDocumentService implements DocumentService {
    @Override
    public String readDocument(String docId) {
        return "Content of " + docId;
    }

    @Override
    public void writeDocument(String docId, String content) {
        System.out.println("Written: " + content);
    }

    @Override
    public void deleteDocument(String docId) {
        System.out.println("Deleted: " + docId);
    }
}

// Protection Proxy — checks permissions
public class DocumentServiceProxy implements DocumentService {
    private final RealDocumentService realService;
    private final User currentUser;

    public DocumentServiceProxy(RealDocumentService realService, User currentUser) {
        this.realService = realService;
        this.currentUser = currentUser;
    }

    @Override
    public String readDocument(String docId) {
        // Everyone can read
        return realService.readDocument(docId);
    }

    @Override
    public void writeDocument(String docId, String content) {
        if (!currentUser.hasRole("EDITOR")) {
            throw new SecurityException("User not authorized to write");
        }
        realService.writeDocument(docId, content);
    }

    @Override
    public void deleteDocument(String docId) {
        if (!currentUser.hasRole("ADMIN")) {
            throw new SecurityException("User not authorized to delete");
        }
        realService.deleteDocument(docId);
    }
}
```

### Real-World Examples
- Spring AOP Proxies — method interception
- JPA/Hibernate Lazy Loading — proxied entity collections
- `java.lang.reflect.Proxy` — dynamic proxies
- RMI stubs — remote object proxies

---

## Structural Patterns — Comparison

| Pattern | Purpose | Key Idea |
|---------|---------|----------|
| **Adapter** | Make incompatible interfaces work | Convert interface A to B |
| **Bridge** | Separate abstraction from implementation | Two independent hierarchies |
| **Composite** | Tree structures | Treat leaf and composite same |
| **Decorator** | Add behavior dynamically | Stackable wrappers |
| **Facade** | Simplify complex subsystem | One unified interface |
| **Flyweight** | Share common state | Memory optimization |
| **Proxy** | Control access to object | Surrogate/placeholder |

### Quick Decision Guide

| Need | Pattern |
|------|---------|
| Third-party integration | Adapter |
| Two varying dimensions | Bridge |
| Tree / hierarchy data | Composite |
| Stack features on object | Decorator |
| Simplify complex API | Facade |
| Too many similar objects | Flyweight |
| Lazy load / access control | Proxy |
