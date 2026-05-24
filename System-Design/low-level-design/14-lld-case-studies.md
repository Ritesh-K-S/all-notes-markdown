# LLD Case Studies — Practice Problems

Real-world LLD interview problems with class design, key patterns used, and core implementation logic.

---

## 1. Parking Lot System

### Requirements
- Multi-floor parking lot with different spot sizes (Small, Medium, Large)
- Support multiple vehicle types (Bike, Car, Truck)
- Assign nearest available spot to a vehicle
- Track entry/exit times and calculate fees

### Key Classes

```java
public enum VehicleType { BIKE, CAR, TRUCK }
public enum SpotSize { SMALL, MEDIUM, LARGE }

public class Vehicle {
    private String licensePlate;
    private VehicleType type;
}

public class ParkingSpot {
    private String spotId;
    private int floor;
    private SpotSize size;
    private boolean isOccupied;
    private Vehicle currentVehicle;

    public boolean canFitVehicle(Vehicle vehicle) {
        return !isOccupied && size.ordinal() >= vehicle.getType().ordinal();
    }

    public void park(Vehicle vehicle) {
        this.currentVehicle = vehicle;
        this.isOccupied = true;
    }

    public void vacate() {
        this.currentVehicle = null;
        this.isOccupied = false;
    }
}

public class ParkingTicket {
    private String ticketId;
    private Vehicle vehicle;
    private ParkingSpot spot;
    private Instant entryTime;
    private Instant exitTime;
    private BigDecimal fee;
}

public class ParkingLot {
    private List<ParkingFloor> floors;
    private FeeCalculator feeCalculator;

    public ParkingTicket parkVehicle(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle);
        if (spot == null) throw new ParkingFullException();
        spot.park(vehicle);
        return new ParkingTicket(vehicle, spot, Instant.now());
    }

    public BigDecimal unparkVehicle(ParkingTicket ticket) {
        ticket.getSpot().vacate();
        ticket.setExitTime(Instant.now());
        BigDecimal fee = feeCalculator.calculate(ticket);
        ticket.setFee(fee);
        return fee;
    }

    private ParkingSpot findAvailableSpot(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            for (ParkingSpot spot : floor.getSpots()) {
                if (spot.canFitVehicle(vehicle)) return spot;
            }
        }
        return null;
    }
}
```

### Patterns Used
- **Strategy** → FeeCalculator (hourly, flat, per-minute)
- **Singleton** → ParkingLot instance
- **Enum** → VehicleType, SpotSize

---

## 2. Elevator System

### Requirements
- Building with N floors and M elevators
- Handle up/down requests from any floor
- Optimize elevator movement (minimize wait time)

### Key Classes

```java
public enum Direction { UP, DOWN, IDLE }

public class Elevator {
    private int id;
    private int currentFloor;
    private Direction direction;
    private TreeSet<Integer> upStops;    // Sorted stops going up
    private TreeSet<Integer> downStops;  // Sorted stops going down

    public void addStop(int floor) {
        if (floor > currentFloor) upStops.add(floor);
        else if (floor < currentFloor) downStops.add(floor);
    }

    public void moveNext() {
        if (direction == Direction.UP) {
            Integer next = upStops.higher(currentFloor);
            if (next != null) {
                currentFloor = next;
                upStops.remove(next);
            } else {
                direction = downStops.isEmpty() ? Direction.IDLE : Direction.DOWN;
            }
        } else if (direction == Direction.DOWN) {
            Integer next = downStops.lower(currentFloor);
            if (next != null) {
                currentFloor = next;
                downStops.remove(next);
            } else {
                direction = upStops.isEmpty() ? Direction.IDLE : Direction.UP;
            }
        }
    }
}

public class ElevatorController {
    private List<Elevator> elevators;

    public Elevator assignElevator(int floor, Direction direction) {
        return elevators.stream()
            .min(Comparator.comparingInt(e -> Math.abs(e.getCurrentFloor() - floor)))
            .orElseThrow(() -> new NoElevatorAvailableException());
    }
}
```

### Patterns Used
- **Strategy** → ElevatorSelectionStrategy (nearest, least loaded)
- **State** → Elevator states (Moving, Idle, Maintenance)
- **Observer** → Floor display listens to elevator position changes

---

## 3. Library Management System

### Requirements
- Members can search, borrow, return, and reserve books
- Track due dates and calculate fines for late returns
- Limit books per member (max 5)

### Key Classes

```java
public class Book {
    private String isbn;
    private String title;
    private String author;
    private List<BookCopy> copies;
}

public class BookCopy {
    private String copyId;
    private Book book;
    private BookStatus status;  // AVAILABLE, BORROWED, RESERVED, LOST
}

public class Member {
    private String memberId;
    private String name;
    private List<BorrowRecord> activeLoans;
    private static final int MAX_BOOKS = 5;

    public boolean canBorrow() {
        return activeLoans.size() < MAX_BOOKS;
    }
}

public class BorrowRecord {
    private BookCopy bookCopy;
    private Member member;
    private LocalDate borrowDate;
    private LocalDate dueDate;
    private LocalDate returnDate;
}

public class LibraryService {
    private final BookRepository bookRepository;
    private final FineCalculator fineCalculator;

    public BorrowRecord borrowBook(String memberId, String isbn) {
        Member member = memberRepository.findById(memberId);
        if (!member.canBorrow()) throw new BorrowLimitExceededException();

        BookCopy copy = bookRepository.findAvailableCopy(isbn);
        if (copy == null) throw new BookNotAvailableException();

        copy.setStatus(BookStatus.BORROWED);
        BorrowRecord record = new BorrowRecord(copy, member,
            LocalDate.now(), LocalDate.now().plusDays(14));
        member.getActiveLoans().add(record);
        return record;
    }

    public BigDecimal returnBook(String borrowRecordId) {
        BorrowRecord record = borrowRepository.findById(borrowRecordId);
        record.setReturnDate(LocalDate.now());
        record.getBookCopy().setStatus(BookStatus.AVAILABLE);
        return fineCalculator.calculate(record);
    }
}
```

---

## 4. Vending Machine

### Requirements
- Supports multiple products with different prices
- Accepts coins/notes, returns change
- Dispenses product after sufficient payment

### Key Classes (State Pattern)

```java
public interface VendingMachineState {
    void selectProduct(VendingMachine machine, Product product);
    void insertMoney(VendingMachine machine, BigDecimal amount);
    void dispense(VendingMachine machine);
    void cancel(VendingMachine machine);
}

public class IdleState implements VendingMachineState {
    public void selectProduct(VendingMachine machine, Product product) {
        if (product.getQuantity() <= 0) throw new OutOfStockException();
        machine.setSelectedProduct(product);
        machine.setState(new ProductSelectedState());
    }
    // Other methods throw InvalidStateException
}

public class ProductSelectedState implements VendingMachineState {
    public void insertMoney(VendingMachine machine, BigDecimal amount) {
        machine.addMoney(amount);
        if (machine.getCurrentBalance().compareTo(
                machine.getSelectedProduct().getPrice()) >= 0) {
            machine.setState(new ReadyToDispenseState());
        }
    }

    public void cancel(VendingMachine machine) {
        machine.refund();
        machine.reset();
        machine.setState(new IdleState());
    }
}

public class ReadyToDispenseState implements VendingMachineState {
    public void dispense(VendingMachine machine) {
        Product product = machine.getSelectedProduct();
        product.decrementQuantity();
        BigDecimal change = machine.getCurrentBalance()
            .subtract(product.getPrice());
        machine.returnChange(change);
        machine.reset();
        machine.setState(new IdleState());
    }
}
```

---

## 5. Notification System

### Requirements
- Send notifications via Email, SMS, Push
- Support priority levels (HIGH, MEDIUM, LOW)
- Retry failed notifications

### Key Classes

```java
// Strategy Pattern for notification channels
public interface NotificationChannel {
    void send(Notification notification);
}

public class EmailChannel implements NotificationChannel { /* ... */ }
public class SmsChannel implements NotificationChannel { /* ... */ }
public class PushChannel implements NotificationChannel { /* ... */ }

// Observer Pattern for notification events
public class NotificationService {
    private final Map<NotificationType, List<NotificationChannel>> channelMap;
    private final RetryHandler retryHandler;

    public void notify(Notification notification) {
        List<NotificationChannel> channels = channelMap.get(notification.getType());
        for (NotificationChannel channel : channels) {
            retryHandler.executeWithRetry(() -> channel.send(notification), 3);
        }
    }
}

// Decorator for adding functionality
public class LoggingNotificationChannel implements NotificationChannel {
    private final NotificationChannel delegate;

    public void send(Notification notification) {
        log.info("Sending notification: {}", notification.getId());
        delegate.send(notification);
        log.info("Notification sent: {}", notification.getId());
    }
}

// Template for user preferences
public class UserNotificationPreference {
    private String userId;
    private Map<NotificationType, List<ChannelType>> preferences;
    private boolean doNotDisturb;
    private LocalTime quietHoursStart;
    private LocalTime quietHoursEnd;
}
```

---

## 6. Rate Limiter (LLD)

### Requirements
- Limit requests per user/IP per time window
- Support different limits for different APIs
- Thread-safe

### Implementation: Token Bucket

```java
public class RateLimiter {
    private final Map<String, UserBucket> buckets = new ConcurrentHashMap<>();
    private final int maxTokens;
    private final int refillPerSecond;

    public boolean allowRequest(String userId) {
        UserBucket bucket = buckets.computeIfAbsent(userId,
            k -> new UserBucket(maxTokens, refillPerSecond));
        return bucket.tryConsume();
    }
}

public class UserBucket {
    private double tokens;
    private final int maxTokens;
    private final int refillPerSecond;
    private long lastRefillNanos;

    public synchronized boolean tryConsume() {
        refill();
        if (tokens >= 1) {
            tokens--;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillNanos) / 1_000_000_000.0;
        tokens = Math.min(maxTokens, tokens + elapsed * refillPerSecond);
        lastRefillNanos = now;
    }
}
```

---

## Quick Reference — Case Study → Key Patterns

| Case Study | Primary Patterns |
|-----------|-----------------|
| Parking Lot | Strategy, Singleton, Factory |
| Elevator System | Strategy, State, Observer |
| Library Management | State (book status), Template Method |
| Vending Machine | State, Factory |
| Notification System | Strategy, Observer, Decorator |
| Rate Limiter | Singleton, Strategy |
| Tic-Tac-Toe | State, Strategy (AI player) |
| BookMyShow | Observer (seat updates), Strategy (pricing) |
| Snake & Ladder | State, Factory, Strategy (dice) |
| ATM Machine | State, Chain of Responsibility (cash dispense) |
