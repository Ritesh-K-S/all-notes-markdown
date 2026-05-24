# Behavioral Design Patterns

Behavioral patterns define **how objects interact and communicate** with each other. They focus on the assignment of responsibilities between objects and the patterns of communication.

---

## 1. Strategy Pattern

> **Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.**

### When to Use
- Multiple algorithms exist for a task
- You want to select the algorithm **at runtime**
- Avoid long `if-else` or `switch` chains for selecting behavior

### Example — Sorting Strategy

```java
// Strategy interface
public interface SortStrategy {
    <T extends Comparable<T>> void sort(List<T> list);
}

// Concrete strategies
public class BubbleSortStrategy implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> list) {
        System.out.println("Sorting using Bubble Sort");
        // Bubble sort implementation
        for (int i = 0; i < list.size() - 1; i++) {
            for (int j = 0; j < list.size() - i - 1; j++) {
                if (list.get(j).compareTo(list.get(j + 1)) > 0) {
                    Collections.swap(list, j, j + 1);
                }
            }
        }
    }
}

public class QuickSortStrategy implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> list) {
        System.out.println("Sorting using Quick Sort");
        Collections.sort(list);  // Simplified
    }
}

public class MergeSortStrategy implements SortStrategy {
    @Override
    public <T extends Comparable<T>> void sort(List<T> list) {
        System.out.println("Sorting using Merge Sort");
        Collections.sort(list);  // Simplified
    }
}

// Context — uses a strategy
public class Sorter {
    private SortStrategy strategy;

    public Sorter(SortStrategy strategy) {
        this.strategy = strategy;
    }

    // Change strategy at runtime
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public <T extends Comparable<T>> void sort(List<T> list) {
        strategy.sort(list);
    }
}

// Usage
List<Integer> data = new ArrayList<>(List.of(5, 2, 8, 1, 9));

Sorter sorter = new Sorter(new BubbleSortStrategy());
sorter.sort(data);  // Uses Bubble Sort

sorter.setStrategy(new QuickSortStrategy());
sorter.sort(data);  // Now uses Quick Sort
```

### Real-World Example — Payment Strategy

```java
public interface PaymentStrategy {
    void pay(double amount);
}

public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid ₹" + amount + " via Credit Card: " + cardNumber);
    }
}

public class UPIPayment implements PaymentStrategy {
    private String upiId;

    public UPIPayment(String upiId) {
        this.upiId = upiId;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid ₹" + amount + " via UPI: " + upiId);
    }
}

public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();

    public void addItem(Item item) { items.add(item); }

    public void checkout(PaymentStrategy paymentStrategy) {
        double total = items.stream().mapToDouble(Item::getPrice).sum();
        paymentStrategy.pay(total);
    }
}
```

---

## 2. Observer Pattern

> **Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.**

Also called: **Publish-Subscribe**, **Event-Listener**

### When to Use
- One object's state change should notify multiple objects
- Event-driven systems
- Loose coupling between event source and listeners

### Example — Stock Price Notification

```java
// Observer interface
public interface StockObserver {
    void update(String stockSymbol, double price);
}

// Subject (Observable)
public class StockMarket {
    private final Map<String, Double> stockPrices = new HashMap<>();
    private final List<StockObserver> observers = new ArrayList<>();

    public void addObserver(StockObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(StockObserver observer) {
        observers.remove(observer);
    }

    public void setStockPrice(String symbol, double price) {
        stockPrices.put(symbol, price);
        notifyObservers(symbol, price);
    }

    private void notifyObservers(String symbol, double price) {
        for (StockObserver observer : observers) {
            observer.update(symbol, price);
        }
    }
}

// Concrete Observers
public class MobileApp implements StockObserver {
    private String userName;

    public MobileApp(String userName) {
        this.userName = userName;
    }

    @Override
    public void update(String stockSymbol, double price) {
        System.out.println("[Mobile - " + userName + "] " + 
                           stockSymbol + " is now ₹" + price);
    }
}

public class WebDashboard implements StockObserver {
    @Override
    public void update(String stockSymbol, double price) {
        System.out.println("[Web Dashboard] Updating " + stockSymbol + " to ₹" + price);
    }
}

public class TradingBot implements StockObserver {
    private double threshold;

    public TradingBot(double threshold) {
        this.threshold = threshold;
    }

    @Override
    public void update(String stockSymbol, double price) {
        if (price < threshold) {
            System.out.println("[Bot] BUY " + stockSymbol + " at ₹" + price);
        } else {
            System.out.println("[Bot] HOLD " + stockSymbol + " at ₹" + price);
        }
    }
}

// Usage
StockMarket market = new StockMarket();
market.addObserver(new MobileApp("Ritesh"));
market.addObserver(new WebDashboard());
market.addObserver(new TradingBot(1500.0));

market.setStockPrice("INFY", 1450.0);  // All three observers notified
market.setStockPrice("TCS", 3200.0);   // All three observers notified
```

### Real-World Examples
- Java's `PropertyChangeListener`
- Spring's `ApplicationEvent` and `@EventListener`
- JavaScript DOM event listeners
- Message queues (Kafka, RabbitMQ)

---

## 3. Command Pattern

> **Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue requests, and support undoable operations.**

### When to Use
- **Undo/Redo** functionality
- **Queue** operations for later execution
- **Log** operations for replay
- Decouple the invoker from the executor

### Example — Text Editor with Undo

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver — the actual object being operated on
public class TextEditor {
    private StringBuilder content = new StringBuilder();

    public void insertText(int position, String text) {
        content.insert(position, text);
    }

    public void deleteText(int position, int length) {
        content.delete(position, position + length);
    }

    public String getContent() {
        return content.toString();
    }
}

// Concrete Commands
public class InsertCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final String text;

    public InsertCommand(TextEditor editor, int position, String text) {
        this.editor = editor;
        this.position = position;
        this.text = text;
    }

    @Override
    public void execute() {
        editor.insertText(position, text);
    }

    @Override
    public void undo() {
        editor.deleteText(position, text.length());
    }
}

public class DeleteCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final int length;
    private String deletedText;  // Save for undo

    public DeleteCommand(TextEditor editor, int position, int length) {
        this.editor = editor;
        this.position = position;
        this.length = length;
    }

    @Override
    public void execute() {
        // Save deleted text before deleting
        deletedText = editor.getContent().substring(position, position + length);
        editor.deleteText(position, length);
    }

    @Override
    public void undo() {
        editor.insertText(position, deletedText);
    }
}

// Invoker — manages command history
public class CommandManager {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void executeCommand(Command command) {
        command.execute();
        history.push(command);
        redoStack.clear();  // Clear redo stack on new command
    }

    public void undo() {
        if (!history.isEmpty()) {
            Command command = history.pop();
            command.undo();
            redoStack.push(command);
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            Command command = redoStack.pop();
            command.execute();
            history.push(command);
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandManager manager = new CommandManager();

manager.executeCommand(new InsertCommand(editor, 0, "Hello "));
manager.executeCommand(new InsertCommand(editor, 6, "World"));
System.out.println(editor.getContent());  // "Hello World"

manager.undo();
System.out.println(editor.getContent());  // "Hello "

manager.redo();
System.out.println(editor.getContent());  // "Hello World"
```

---

## 4. Chain of Responsibility Pattern

> **Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until one handles it.**

### When to Use
- Multiple handlers can process a request
- Handler is determined **at runtime**
- Request should pass through a **pipeline** of processing steps
- Logging levels, authentication filters, validation chains

### Example — Support Ticket Escalation

```java
// Handler
public abstract class SupportHandler {
    private SupportHandler nextHandler;

    public SupportHandler setNext(SupportHandler handler) {
        this.nextHandler = handler;
        return handler;  // For chaining
    }

    public void handle(SupportTicket ticket) {
        if (canHandle(ticket)) {
            processTicket(ticket);
        } else if (nextHandler != null) {
            nextHandler.handle(ticket);
        } else {
            System.out.println("No handler found for: " + ticket.getDescription());
        }
    }

    protected abstract boolean canHandle(SupportTicket ticket);
    protected abstract void processTicket(SupportTicket ticket);
}

public class SupportTicket {
    public enum Priority { LOW, MEDIUM, HIGH, CRITICAL }

    private String description;
    private Priority priority;

    public SupportTicket(String description, Priority priority) {
        this.description = description;
        this.priority = priority;
    }

    public String getDescription() { return description; }
    public Priority getPriority() { return priority; }
}

// Concrete Handlers
public class Level1Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority() == SupportTicket.Priority.LOW;
    }

    @Override
    protected void processTicket(SupportTicket ticket) {
        System.out.println("[L1 Support] Handling: " + ticket.getDescription());
    }
}

public class Level2Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority() == SupportTicket.Priority.MEDIUM;
    }

    @Override
    protected void processTicket(SupportTicket ticket) {
        System.out.println("[L2 Support] Handling: " + ticket.getDescription());
    }
}

public class Level3Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority() == SupportTicket.Priority.HIGH || 
               ticket.getPriority() == SupportTicket.Priority.CRITICAL;
    }

    @Override
    protected void processTicket(SupportTicket ticket) {
        System.out.println("[L3 Support / Manager] Handling: " + ticket.getDescription());
    }
}

// Usage — build chain
SupportHandler l1 = new Level1Support();
SupportHandler l2 = new Level2Support();
SupportHandler l3 = new Level3Support();

l1.setNext(l2).setNext(l3);  // L1 → L2 → L3

l1.handle(new SupportTicket("Password reset", SupportTicket.Priority.LOW));       // L1
l1.handle(new SupportTicket("App crash", SupportTicket.Priority.MEDIUM));         // L2
l1.handle(new SupportTicket("Data breach", SupportTicket.Priority.CRITICAL));     // L3
```

### Real-World Examples
- Servlet Filters in Java EE
- Spring Security Filter Chain
- Exception handling (try-catch chain)
- Middleware in Express.js / Spring Interceptors

---

## 5. State Pattern

> **Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.**

### When to Use
- Object behavior depends on its **current state**
- Large `if-else` or `switch` on state
- State transitions have well-defined rules

### Example — Vending Machine States

```java
// State interface
public interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void selectProduct(VendingMachine machine);
    void dispense(VendingMachine machine);
}

// Context
public class VendingMachine {
    private VendingMachineState currentState;
    private int inventory;

    // States
    private final VendingMachineState idleState = new IdleState();
    private final VendingMachineState hasMoneyState = new HasMoneyState();
    private final VendingMachineState dispensingState = new DispensingState();
    private final VendingMachineState outOfStockState = new OutOfStockState();

    public VendingMachine(int inventory) {
        this.inventory = inventory;
        this.currentState = inventory > 0 ? idleState : outOfStockState;
    }

    public void setState(VendingMachineState state) { this.currentState = state; }
    public int getInventory() { return inventory; }
    public void decrementInventory() { inventory--; }

    // State getters
    public VendingMachineState getIdleState() { return idleState; }
    public VendingMachineState getHasMoneyState() { return hasMoneyState; }
    public VendingMachineState getDispensingState() { return dispensingState; }
    public VendingMachineState getOutOfStockState() { return outOfStockState; }

    // Delegate to current state
    public void insertCoin() { currentState.insertCoin(this); }
    public void selectProduct() { currentState.selectProduct(this); }
    public void dispense() { currentState.dispense(this); }
}

// Concrete States
public class IdleState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin inserted");
        machine.setState(machine.getHasMoneyState());
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Please insert coin first");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert coin and select product");
    }
}

public class HasMoneyState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin already inserted");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Product selected");
        machine.setState(machine.getDispensingState());
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first");
    }
}

public class DispensingState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Please wait, dispensing...");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Already dispensing...");
    }

    @Override
    public void dispense(VendingMachine machine) {
        machine.decrementInventory();
        System.out.println("Product dispensed!");
        if (machine.getInventory() > 0) {
            machine.setState(machine.getIdleState());
        } else {
            System.out.println("Out of stock!");
            machine.setState(machine.getOutOfStockState());
        }
    }
}

public class OutOfStockState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Machine is out of stock. Returning coin.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Machine is out of stock.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Machine is out of stock.");
    }
}
```

---

## 6. Template Method Pattern

> **Define the skeleton of an algorithm in a method, deferring some steps to subclasses. Subclasses can override specific steps without changing the algorithm's structure.**

### When to Use
- Multiple classes share the **same algorithm structure** but differ in specific steps
- You want to enforce a **fixed sequence** of operations
- "Don't call us, we'll call you" (Hollywood Principle)

### Example — Data Processing Pipeline

```java
// Abstract class with template method
public abstract class DataProcessor {

    // Template method — defines the algorithm skeleton (FINAL — cannot be overridden)
    public final void process() {
        readData();
        parseData();
        validateData();
        transformData();
        saveData();
        
        // Hook method — optional step
        if (shouldNotify()) {
            notifyCompletion();
        }
    }

    // Abstract steps — subclasses MUST implement
    protected abstract void readData();
    protected abstract void parseData();
    protected abstract void transformData();
    protected abstract void saveData();

    // Concrete step — common to all subclasses
    protected void validateData() {
        System.out.println("Validating data (common logic)...");
    }

    // Hook method — subclasses CAN override (optional)
    protected boolean shouldNotify() {
        return false;
    }

    protected void notifyCompletion() {
        System.out.println("Processing completed notification sent");
    }
}

// Concrete implementations
public class CSVDataProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV file...");
    }

    @Override
    protected void parseData() {
        System.out.println("Parsing CSV rows...");
    }

    @Override
    protected void transformData() {
        System.out.println("Transforming CSV data to objects...");
    }

    @Override
    protected void saveData() {
        System.out.println("Saving to database...");
    }
}

public class JSONDataProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading JSON file...");
    }

    @Override
    protected void parseData() {
        System.out.println("Parsing JSON nodes...");
    }

    @Override
    protected void transformData() {
        System.out.println("Transforming JSON to objects...");
    }

    @Override
    protected void saveData() {
        System.out.println("Saving to NoSQL database...");
    }

    @Override
    protected boolean shouldNotify() {
        return true;  // JSON processor sends notification
    }
}

// Usage
DataProcessor csvProcessor = new CSVDataProcessor();
csvProcessor.process();  // Follows the exact same sequence

DataProcessor jsonProcessor = new JSONDataProcessor();
jsonProcessor.process();  // Same sequence, different steps + notification
```

### Template Method vs Strategy

| Template Method | Strategy |
|----------------|----------|
| Uses inheritance | Uses composition |
| Algorithm structure is fixed | Entire algorithm is swappable |
| Subclass changes steps | Client chooses strategy |
| Compile-time decision | Runtime decision |

---

## 7. Iterator Pattern

> **Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.**

### When to Use
- Traverse a collection without exposing its internal structure
- Support multiple traversal strategies
- Provide a uniform interface for traversing different collection types

### Example — Custom Collection with Iterator

```java
// Iterator interface
public interface Iterator<T> {
    boolean hasNext();
    T next();
}

// Collection interface
public interface IterableCollection<T> {
    Iterator<T> createIterator();
}

// Concrete Collection — custom linked list
public class BookCollection implements IterableCollection<Book> {
    private List<Book> books = new ArrayList<>();

    public void addBook(Book book) {
        books.add(book);
    }

    public int size() {
        return books.size();
    }

    public Book getAt(int index) {
        return books.get(index);
    }

    @Override
    public Iterator<Book> createIterator() {
        return new BookIterator(this);
    }

    // Can also provide a filtered iterator
    public Iterator<Book> createGenreIterator(String genre) {
        return new GenreFilterIterator(this, genre);
    }
}

// Concrete Iterator
public class BookIterator implements Iterator<Book> {
    private final BookCollection collection;
    private int currentIndex = 0;

    public BookIterator(BookCollection collection) {
        this.collection = collection;
    }

    @Override
    public boolean hasNext() {
        return currentIndex < collection.size();
    }

    @Override
    public Book next() {
        Book book = collection.getAt(currentIndex);
        currentIndex++;
        return book;
    }
}

// Filtered Iterator — only returns books of a specific genre
public class GenreFilterIterator implements Iterator<Book> {
    private final BookCollection collection;
    private final String genre;
    private int currentIndex = 0;

    public GenreFilterIterator(BookCollection collection, String genre) {
        this.collection = collection;
        this.genre = genre;
    }

    @Override
    public boolean hasNext() {
        while (currentIndex < collection.size()) {
            if (collection.getAt(currentIndex).getGenre().equals(genre)) {
                return true;
            }
            currentIndex++;
        }
        return false;
    }

    @Override
    public Book next() {
        Book book = collection.getAt(currentIndex);
        currentIndex++;
        return book;
    }
}

// Usage
BookCollection library = new BookCollection();
library.addBook(new Book("Clean Code", "Tech"));
library.addBook(new Book("Harry Potter", "Fiction"));
library.addBook(new Book("Design Patterns", "Tech"));

// Iterate all books
Iterator<Book> all = library.createIterator();
while (all.hasNext()) {
    System.out.println(all.next());
}

// Iterate only tech books
Iterator<Book> techBooks = library.createGenreIterator("Tech");
while (techBooks.hasNext()) {
    System.out.println(techBooks.next());
}
```

### Real-World in Java
- `java.util.Iterator` interface
- Enhanced for-loop uses `Iterable` / `Iterator` internally
- `Stream` API — a modern iterator

---

## 8. Mediator Pattern

> **Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly.**

### When to Use
- Many objects communicate with each other in complex ways
- You want to reduce **direct dependencies** between communicating objects
- Chat rooms, air traffic control, UI form components

### Example — Chat Room

```java
// Mediator interface
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// Colleague
public abstract class User {
    protected ChatMediator mediator;
    protected String name;

    public User(ChatMediator mediator, String name) {
        this.mediator = mediator;
        this.name = name;
    }

    public String getName() { return name; }

    public abstract void send(String message);
    public abstract void receive(String message);
}

// Concrete Mediator
public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    @Override
    public void addUser(User user) {
        users.add(user);
    }

    @Override
    public void sendMessage(String message, User sender) {
        for (User user : users) {
            // Don't send to the sender
            if (user != sender) {
                user.receive(message);
            }
        }
    }
}

// Concrete Colleague
public class ChatUser extends User {

    public ChatUser(ChatMediator mediator, String name) {
        super(mediator, name);
    }

    @Override
    public void send(String message) {
        System.out.println(name + " sends: " + message);
        mediator.sendMessage(message, this);
    }

    @Override
    public void receive(String message) {
        System.out.println(name + " received: " + message);
    }
}

// Usage
ChatMediator chatRoom = new ChatRoom();

User alice = new ChatUser(chatRoom, "Alice");
User bob = new ChatUser(chatRoom, "Bob");
User charlie = new ChatUser(chatRoom, "Charlie");

chatRoom.addUser(alice);
chatRoom.addUser(bob);
chatRoom.addUser(charlie);

alice.send("Hello everyone!");
// Bob received: Hello everyone!
// Charlie received: Hello everyone!
```

---

## 9. Memento Pattern

> **Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.**

### When to Use
- **Undo/Redo** operations
- **Snapshot/Checkpoint** functionality
- Save and restore state (game save)

### Example — Game Save System

```java
// Memento — stores state snapshot
public class GameMemento {
    private final int level;
    private final int health;
    private final int score;
    private final String checkpoint;

    public GameMemento(int level, int health, int score, String checkpoint) {
        this.level = level;
        this.health = health;
        this.score = score;
        this.checkpoint = checkpoint;
    }

    // Only accessible by Originator
    int getLevel() { return level; }
    int getHealth() { return health; }
    int getScore() { return score; }
    String getCheckpoint() { return checkpoint; }
}

// Originator — creates and restores from mementos
public class Game {
    private int level;
    private int health;
    private int score;
    private String checkpoint;

    public Game() {
        this.level = 1;
        this.health = 100;
        this.score = 0;
        this.checkpoint = "Start";
    }

    public void play(String checkpointName, int scoreGained, int healthLost) {
        this.checkpoint = checkpointName;
        this.score += scoreGained;
        this.health -= healthLost;
        System.out.println("Playing... " + this);
    }

    public void levelUp() {
        this.level++;
        System.out.println("Level Up! Now at level " + level);
    }

    // Save current state
    public GameMemento save() {
        System.out.println("Game saved at: " + checkpoint);
        return new GameMemento(level, health, score, checkpoint);
    }

    // Restore from memento
    public void restore(GameMemento memento) {
        this.level = memento.getLevel();
        this.health = memento.getHealth();
        this.score = memento.getScore();
        this.checkpoint = memento.getCheckpoint();
        System.out.println("Game restored to: " + checkpoint);
    }

    @Override
    public String toString() {
        return "Game[level=" + level + ", health=" + health + 
               ", score=" + score + ", checkpoint=" + checkpoint + "]";
    }
}

// Caretaker — manages memento history
public class GameSaveManager {
    private final Deque<GameMemento> saves = new ArrayDeque<>();

    public void save(GameMemento memento) {
        saves.push(memento);
    }

    public GameMemento loadLastSave() {
        if (saves.isEmpty()) {
            throw new IllegalStateException("No saves available");
        }
        return saves.pop();
    }
}

// Usage
Game game = new Game();
GameSaveManager saveManager = new GameSaveManager();

game.play("Forest", 100, 20);
saveManager.save(game.save());          // Save checkpoint

game.play("Castle", 200, 50);
game.levelUp();
saveManager.save(game.save());          // Save another checkpoint

game.play("Dragon Fight", 500, 80);     // Oops, health too low
System.out.println("Current: " + game);

game.restore(saveManager.loadLastSave()); // Restore to Castle
System.out.println("After restore: " + game);
```

---

## 10. Visitor Pattern

> **Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.**

### When to Use
- Add new operations to existing class hierarchies **without modifying** them
- The class hierarchy is stable but operations change frequently
- Double dispatch is needed

### Example — Shopping Cart with Different Operations

```java
// Element interface
public interface ItemElement {
    void accept(ShoppingCartVisitor visitor);
}

// Concrete Elements
public class Book implements ItemElement {
    private String title;
    private double price;

    public Book(String title, double price) {
        this.title = title;
        this.price = price;
    }

    public String getTitle() { return title; }
    public double getPrice() { return price; }

    @Override
    public void accept(ShoppingCartVisitor visitor) {
        visitor.visit(this);  // Double dispatch
    }
}

public class Electronic implements ItemElement {
    private String name;
    private double price;
    private double weight;

    public Electronic(String name, double price, double weight) {
        this.name = name;
        this.price = price;
        this.weight = weight;
    }

    public String getName() { return name; }
    public double getPrice() { return price; }
    public double getWeight() { return weight; }

    @Override
    public void accept(ShoppingCartVisitor visitor) {
        visitor.visit(this);
    }
}

// Visitor interface
public interface ShoppingCartVisitor {
    void visit(Book book);
    void visit(Electronic electronic);
}

// Concrete Visitors — different operations
public class PriceCalculatorVisitor implements ShoppingCartVisitor {
    private double totalPrice = 0;

    @Override
    public void visit(Book book) {
        // Books get 10% discount
        totalPrice += book.getPrice() * 0.9;
    }

    @Override
    public void visit(Electronic electronic) {
        // Electronics have shipping cost based on weight
        totalPrice += electronic.getPrice() + (electronic.getWeight() * 5);
    }

    public double getTotalPrice() { return totalPrice; }
}

public class TaxCalculatorVisitor implements ShoppingCartVisitor {
    private double totalTax = 0;

    @Override
    public void visit(Book book) {
        totalTax += book.getPrice() * 0.05;  // 5% tax on books
    }

    @Override
    public void visit(Electronic electronic) {
        totalTax += electronic.getPrice() * 0.18;  // 18% tax on electronics
    }

    public double getTotalTax() { return totalTax; }
}

// Usage
List<ItemElement> cart = List.of(
    new Book("Design Patterns", 500),
    new Book("Clean Code", 400),
    new Electronic("Laptop", 50000, 2.5)
);

PriceCalculatorVisitor priceCalc = new PriceCalculatorVisitor();
TaxCalculatorVisitor taxCalc = new TaxCalculatorVisitor();

for (ItemElement item : cart) {
    item.accept(priceCalc);
    item.accept(taxCalc);
}

System.out.println("Total Price: ₹" + priceCalc.getTotalPrice());
System.out.println("Total Tax: ₹" + taxCalc.getTotalTax());
```

---

## Behavioral Patterns — Comparison

| Pattern | Purpose | Key Feature |
|---------|---------|-------------|
| **Strategy** | Swap algorithms | Interchangeable behaviors |
| **Observer** | Event notification | One-to-many updates |
| **Command** | Encapsulate request | Undo/redo, queuing |
| **Chain of Responsibility** | Pass request along chain | Handler pipeline |
| **State** | Change behavior with state | State machine |
| **Template Method** | Fixed algorithm, flexible steps | Inheritance-based |
| **Iterator** | Sequential access | Traverse without exposing internals |
| **Mediator** | Centralized communication | Reduce coupling |
| **Memento** | Save/restore state | Snapshot without breaking encapsulation |
| **Visitor** | Add operations to hierarchy | Double dispatch |

### Quick Decision Guide

| Need | Pattern |
|------|---------|
| Swap algorithm at runtime | Strategy |
| Notify multiple objects of change | Observer |
| Undo/redo support | Command + Memento |
| Pipeline of handlers | Chain of Responsibility |
| Behavior changes with state | State |
| Same steps, different details | Template Method |
| Traverse custom collection | Iterator |
| Many-to-many communication | Mediator |
| Save object snapshots | Memento |
| New operations on stable hierarchy | Visitor |
