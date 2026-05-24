# Enums, Constants & State Machines in LLD

Enums and state machines are fundamental tools for modeling fixed sets of values, statuses, and state transitions in Low Level Design.

---

## 1. Enums in Java

Enums represent a **fixed set of constants**. They are type-safe, can have fields, methods, and implement interfaces.

### Basic Enum

```java
public enum OrderStatus {
    PLACED,
    CONFIRMED,
    SHIPPED,
    DELIVERED,
    CANCELLED,
    RETURNED
}

// Usage
OrderStatus status = OrderStatus.PLACED;

// Iterating all values
for (OrderStatus s : OrderStatus.values()) {
    System.out.println(s);
}

// String to enum
OrderStatus fromString = OrderStatus.valueOf("SHIPPED");

// Enum in switch
switch (status) {
    case PLACED -> System.out.println("Order placed");
    case SHIPPED -> System.out.println("On the way");
    case DELIVERED -> System.out.println("Delivered");
    default -> System.out.println("Other status");
}
```

### Enum with Fields and Methods

```java
public enum PaymentMethod {
    CREDIT_CARD("Credit Card", 2.5),
    DEBIT_CARD("Debit Card", 1.0),
    UPI("UPI", 0.0),
    NET_BANKING("Net Banking", 1.5),
    WALLET("Digital Wallet", 0.5);

    private final String displayName;
    private final double processingFeePercent;

    PaymentMethod(String displayName, double processingFeePercent) {
        this.displayName = displayName;
        this.processingFeePercent = processingFeePercent;
    }

    public String getDisplayName() { return displayName; }

    public double getProcessingFeePercent() { return processingFeePercent; }

    public double calculateFee(double amount) {
        return amount * processingFeePercent / 100;
    }
}

// Usage
PaymentMethod method = PaymentMethod.UPI;
System.out.println(method.getDisplayName());           // "UPI"
System.out.println(method.calculateFee(1000));         // 0.0
System.out.println(PaymentMethod.CREDIT_CARD.calculateFee(1000));  // 25.0
```

### Enum with Abstract Methods

Each constant provides its own implementation — polymorphic behavior.

```java
public enum Discount {
    REGULAR {
        @Override
        public double apply(double price) {
            return price * 0.95;  // 5% off
        }
    },
    PREMIUM {
        @Override
        public double apply(double price) {
            return price * 0.85;  // 15% off
        }
    },
    VIP {
        @Override
        public double apply(double price) {
            return price * 0.75;  // 25% off
        }
    };

    public abstract double apply(double price);
}

// Usage
double finalPrice = Discount.VIP.apply(1000);  // 750.0
```

### Enum Implementing Interface

```java
public interface Printable {
    String toPrintFormat();
}

public enum Priority implements Printable {
    LOW(1, "Low Priority"),
    MEDIUM(2, "Medium Priority"),
    HIGH(3, "High Priority"),
    CRITICAL(4, "Critical Priority");

    private final int level;
    private final String description;

    Priority(int level, String description) {
        this.level = level;
        this.description = description;
    }

    public int getLevel() { return level; }

    @Override
    public String toPrintFormat() {
        return "[" + level + "] " + description;
    }

    // Lookup by level
    public static Priority fromLevel(int level) {
        for (Priority p : values()) {
            if (p.level == level) return p;
        }
        throw new IllegalArgumentException("Invalid priority level: " + level);
    }
}
```

---

## 2. Constants Best Practices

### Using Enum vs static final

```java
// BAD — String constants (no type safety)
public class Status {
    public static final String ACTIVE = "ACTIVE";
    public static final String INACTIVE = "INACTIVE";
    public static final String DELETED = "DELETED";
}

// Anyone can pass any string — no compile-time check
public void setStatus(String status) { }  // Can pass "BLAH" — no error

// GOOD — Enum (type-safe)
public enum Status {
    ACTIVE, INACTIVE, DELETED
}

public void setStatus(Status status) { }  // Only valid values accepted
```

### Constants Class for Configuration

```java
public final class AppConstants {
    private AppConstants() {}  // Prevent instantiation

    public static final int MAX_RETRY_COUNT = 3;
    public static final long SESSION_TIMEOUT_MS = 30 * 60 * 1000L;  // 30 minutes
    public static final String DEFAULT_CURRENCY = "INR";
    public static final int PAGINATION_DEFAULT_SIZE = 20;
    public static final int PAGINATION_MAX_SIZE = 100;
}
```

### When to Use Enum vs Constants

| Use Case | Use |
|----------|-----|
| Fixed set of types/categories | Enum |
| Status/state values | Enum |
| Configuration values | static final constants |
| Numeric limits/thresholds | static final constants |
| Values that need behavior | Enum with methods |

---

## 3. State Machine Design

A state machine models an entity that transitions between well-defined **states** based on **events/actions**.

### Components
- **States** — The possible conditions of the entity
- **Events/Triggers** — What causes transitions
- **Transitions** — Rules defining valid state changes
- **Actions** — What happens during a transition

### Example — Order State Machine

```
                  ┌──────────┐
                  │  PLACED  │
                  └─────┬────┘
                        │ confirm()
                        ▼
                  ┌──────────┐
          ┌───── │ CONFIRMED │ ──────┐
          │      └─────┬────┘       │
          │ cancel()   │ ship()     │ cancel()
          ▼            ▼            │
   ┌───────────┐ ┌──────────┐      │
   │ CANCELLED │ │ SHIPPED  │ ◄────┘
   └───────────┘ └─────┬────┘
                       │ deliver()
                       ▼
                 ┌───────────┐
                 │ DELIVERED │
                 └─────┬────┘
                       │ return()
                       ▼
                 ┌───────────┐
                 │ RETURNED  │
                 └───────────┘
```

### Implementation — Enum with Transitions

```java
public enum OrderStatus {
    PLACED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            return switch (event) {
                case CONFIRM -> CONFIRMED;
                case CANCEL -> CANCELLED;
                default -> throw new InvalidTransitionException(this, event);
            };
        }
    },
    CONFIRMED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            return switch (event) {
                case SHIP -> SHIPPED;
                case CANCEL -> CANCELLED;
                default -> throw new InvalidTransitionException(this, event);
            };
        }
    },
    SHIPPED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            return switch (event) {
                case DELIVER -> DELIVERED;
                default -> throw new InvalidTransitionException(this, event);
            };
        }
    },
    DELIVERED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            return switch (event) {
                case RETURN -> RETURNED;
                default -> throw new InvalidTransitionException(this, event);
            };
        }
    },
    CANCELLED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            throw new InvalidTransitionException(this, event);  // Terminal state
        }
    },
    RETURNED {
        @Override
        public OrderStatus nextState(OrderEvent event) {
            throw new InvalidTransitionException(this, event);  // Terminal state
        }
    };

    public abstract OrderStatus nextState(OrderEvent event);
}

public enum OrderEvent {
    CONFIRM, SHIP, DELIVER, CANCEL, RETURN
}

public class InvalidTransitionException extends RuntimeException {
    public InvalidTransitionException(OrderStatus from, OrderEvent event) {
        super("Invalid transition: " + from + " + " + event);
    }
}
```

### Usage in Service Layer

```java
public class OrderService {

    public void confirmOrder(Order order) {
        OrderStatus newStatus = order.getStatus().nextState(OrderEvent.CONFIRM);
        order.setStatus(newStatus);
        order.setUpdatedAt(LocalDateTime.now());
        orderRepository.save(order);
    }

    public void shipOrder(Order order) {
        OrderStatus newStatus = order.getStatus().nextState(OrderEvent.SHIP);
        order.setStatus(newStatus);
        order.setShippedAt(LocalDateTime.now());
        orderRepository.save(order);
    }

    public void cancelOrder(Order order) {
        OrderStatus newStatus = order.getStatus().nextState(OrderEvent.CANCEL);
        order.setStatus(newStatus);
        order.setCancelledAt(LocalDateTime.now());
        orderRepository.save(order);
        // Trigger refund
    }
}
```

---

## 4. State Machine with Transition Table

An alternative approach using a transition table for more complex state machines.

```java
public class StateMachine<S extends Enum<S>, E extends Enum<E>> {
    
    private final Map<S, Map<E, S>> transitions = new HashMap<>();
    private S currentState;

    public StateMachine(S initialState) {
        this.currentState = initialState;
    }

    public void addTransition(S fromState, E event, S toState) {
        transitions.computeIfAbsent(fromState, k -> new HashMap<>()).put(event, toState);
    }

    public void fire(E event) {
        Map<E, S> stateTransitions = transitions.get(currentState);
        if (stateTransitions == null || !stateTransitions.containsKey(event)) {
            throw new IllegalStateException(
                "No transition from " + currentState + " on event " + event
            );
        }
        S newState = stateTransitions.get(event);
        System.out.println(currentState + " --[" + event + "]--> " + newState);
        currentState = newState;
    }

    public S getCurrentState() {
        return currentState;
    }
}

// Usage — define transitions declaratively
StateMachine<OrderStatus, OrderEvent> sm = new StateMachine<>(OrderStatus.PLACED);

sm.addTransition(OrderStatus.PLACED, OrderEvent.CONFIRM, OrderStatus.CONFIRMED);
sm.addTransition(OrderStatus.PLACED, OrderEvent.CANCEL, OrderStatus.CANCELLED);
sm.addTransition(OrderStatus.CONFIRMED, OrderEvent.SHIP, OrderStatus.SHIPPED);
sm.addTransition(OrderStatus.CONFIRMED, OrderEvent.CANCEL, OrderStatus.CANCELLED);
sm.addTransition(OrderStatus.SHIPPED, OrderEvent.DELIVER, OrderStatus.DELIVERED);
sm.addTransition(OrderStatus.DELIVERED, OrderEvent.RETURN, OrderStatus.RETURNED);

sm.fire(OrderEvent.CONFIRM);   // PLACED --> CONFIRMED
sm.fire(OrderEvent.SHIP);      // CONFIRMED --> SHIPPED
sm.fire(OrderEvent.DELIVER);   // SHIPPED --> DELIVERED
```

---

## 5. Common State Machines in LLD

### Payment Status

```
INITIATED → PROCESSING → SUCCESS
                       → FAILED → RETRYING → SUCCESS
                                            → FAILED
SUCCESS → REFUND_INITIATED → REFUNDED
```

### Booking Status

```
PENDING → CONFIRMED → CHECKED_IN → CHECKED_OUT
        → CANCELLED
CONFIRMED → CANCELLED (with conditions)
```

### User Account Status

```
PENDING_VERIFICATION → ACTIVE → SUSPENDED → ACTIVE
                               → DEACTIVATED
ACTIVE → DEACTIVATED
PENDING_VERIFICATION → EXPIRED
```

### Task/Ticket Status

```
OPEN → IN_PROGRESS → IN_REVIEW → DONE
     → BLOCKED → IN_PROGRESS
IN_REVIEW → IN_PROGRESS (rejected)
Any → CLOSED
```

---

## 6. Enum Best Practices for LLD

### 1. Always Implement toString() for Display

```java
public enum RoomType {
    SINGLE("Single Room", 1),
    DOUBLE("Double Room", 2),
    SUITE("Suite", 4),
    DELUXE("Deluxe Room", 2);

    private final String displayName;
    private final int maxOccupancy;

    RoomType(String displayName, int maxOccupancy) {
        this.displayName = displayName;
        this.maxOccupancy = maxOccupancy;
    }

    @Override
    public String toString() {
        return displayName;
    }
}
```

### 2. Add Lookup Methods

```java
public enum Currency {
    INR("Indian Rupee", "₹"),
    USD("US Dollar", "$"),
    EUR("Euro", "€"),
    GBP("British Pound", "£");

    private final String fullName;
    private final String symbol;
    
    private static final Map<String, Currency> BY_SYMBOL = new HashMap<>();
    
    static {
        for (Currency c : values()) {
            BY_SYMBOL.put(c.symbol, c);
        }
    }

    Currency(String fullName, String symbol) {
        this.fullName = fullName;
        this.symbol = symbol;
    }

    public static Currency fromSymbol(String symbol) {
        Currency currency = BY_SYMBOL.get(symbol);
        if (currency == null) {
            throw new IllegalArgumentException("Unknown currency symbol: " + symbol);
        }
        return currency;
    }
}
```

### 3. Use EnumSet and EnumMap for Performance

```java
// EnumSet — highly efficient set of enums
Set<OrderStatus> activeStatuses = EnumSet.of(
    OrderStatus.PLACED, OrderStatus.CONFIRMED, OrderStatus.SHIPPED
);

Set<OrderStatus> terminalStatuses = EnumSet.of(
    OrderStatus.DELIVERED, OrderStatus.CANCELLED, OrderStatus.RETURNED
);

// Check if status is terminal
if (terminalStatuses.contains(order.getStatus())) {
    System.out.println("Order is in terminal state");
}

// EnumMap — optimized map with enum keys
Map<OrderStatus, String> statusMessages = new EnumMap<>(OrderStatus.class);
statusMessages.put(OrderStatus.PLACED, "Your order has been placed");
statusMessages.put(OrderStatus.SHIPPED, "Your order is on the way");
statusMessages.put(OrderStatus.DELIVERED, "Your order has been delivered");
```

---

## Summary

| Concept | When to Use |
|---------|-------------|
| **Basic Enum** | Fixed set of values (types, categories) |
| **Enum with fields** | Values need metadata (display name, code) |
| **Enum with methods** | Values need behavior (calculate, format) |
| **Enum with abstract method** | Each value has different behavior |
| **Constants class** | Numeric limits, config values |
| **State Machine (enum)** | Entity lifecycle with transitions |
| **State Machine (table)** | Complex/configurable transitions |
| **EnumSet / EnumMap** | High-performance enum collections |
