# 01 - Parking Lot System

## 📋 Problem Statement

Design an automated parking lot system that can:
- Support multiple floors and different vehicle types
- Assign the nearest available spot to a vehicle
- Handle entry/exit and calculate parking fees
- Display available spots per floor

---

## 📌 Requirements

### Functional Requirements
1. The parking lot has multiple **floors**, each with multiple **parking spots**
2. Support vehicle types: **Car, Bike, Truck**
3. Spot types: **Small (Bike), Medium (Car), Large (Truck)**
4. Vehicle gets the **nearest available spot** (closest to entry)
5. On exit, **fee is calculated** based on duration
6. Display board shows **available spots per floor per type**
7. Multiple **entry and exit points**

### Non-Functional Requirements
- Handle **concurrent** entry/exit
- System should be **extensible** for new vehicle types
- Thread-safe spot assignment

---

## 🧩 Core Entities

```
ParkingLot, ParkingFloor, ParkingSpot, Vehicle, Ticket, 
EntryPanel, ExitPanel, DisplayBoard, Payment, ParkingRate
```

---

## 📐 Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         ParkingLot                              │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ (Singleton)                                │
│  - name: String                                                 │
│  - floors: List<ParkingFloor>                                   │
│  - entryPanels: List<EntryPanel>                                │
│  - exitPanels: List<ExitPanel>                                  │
│  + getInstance(): ParkingLot                                    │
│  + addFloor(floor): void                                        │
│  + assignSpot(vehicle): Ticket                                  │
│  + freeSpot(ticket): Payment                                    │
└───────────────┬─────────────────────────────────────────────────┘
                │ 1..*
┌───────────────▼─────────────────────────────────────────────────┐
│                       ParkingFloor                              │
│  - floorNumber: int                                             │
│  - spots: Map<SpotType, List<ParkingSpot>>                      │
│  - displayBoard: DisplayBoard                                   │
│  + getAvailableSpot(vehicleType): ParkingSpot                   │
│  + updateDisplayBoard(): void                                   │
└───────────────┬─────────────────────────────────────────────────┘
                │ 1..*
┌───────────────▼─────────────────────────────────────────────────┐
│                       ParkingSpot                               │
│  - spotId: String                                               │
│  - spotType: SpotType                                           │
│  - isAvailable: boolean                                         │
│  - vehicle: Vehicle                                             │
│  + assignVehicle(vehicle): void                                 │
│  + removeVehicle(): void                                        │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│     Vehicle       │    │  VehicleType     │
│  - licensePlate   │    │  ─ ─ ─ ─ ─ ─    │
│  - type: VType    │    │  CAR             │
│  - color: String  │    │  BIKE            │
│                   │    │  TRUCK           │
└──────────────────┘    └──────────────────┘

┌──────────────────┐    ┌──────────────────┐
│     Ticket        │    │   SpotType       │
│  - ticketId       │    │  ─ ─ ─ ─ ─ ─    │
│  - entryTime      │    │  SMALL           │
│  - vehicle        │    │  MEDIUM          │
│  - spot           │    │  LARGE           │
│  - status         │    │                  │
└──────────────────┘    └──────────────────┘
```

---

## 🔧 Enums and Constants

```java
public enum VehicleType {
    BIKE, CAR, TRUCK
}

public enum SpotType {
    SMALL, MEDIUM, LARGE
}

public enum TicketStatus {
    ACTIVE, PAID, LOST
}

public enum PaymentMethod {
    CASH, CARD, UPI
}
```

---

## 💻 Code Implementation

### Vehicle Class

```java
public abstract class Vehicle {
    private String licensePlate;
    private VehicleType type;
    private String color;

    public Vehicle(String licensePlate, VehicleType type, String color) {
        this.licensePlate = licensePlate;
        this.type = type;
        this.color = color;
    }

    // Getters
    public VehicleType getType() { return type; }
    public String getLicensePlate() { return licensePlate; }
}

public class Car extends Vehicle {
    public Car(String licensePlate, String color) {
        super(licensePlate, VehicleType.CAR, color);
    }
}

public class Bike extends Vehicle {
    public Bike(String licensePlate, String color) {
        super(licensePlate, VehicleType.BIKE, color);
    }
}

public class Truck extends Vehicle {
    public Truck(String licensePlate, String color) {
        super(licensePlate, VehicleType.TRUCK, color);
    }
}
```

### ParkingSpot Class

```java
public class ParkingSpot {
    private String spotId;
    private SpotType spotType;
    private boolean isAvailable;
    private Vehicle vehicle;

    public ParkingSpot(String spotId, SpotType spotType) {
        this.spotId = spotId;
        this.spotType = spotType;
        this.isAvailable = true;
    }

    public synchronized boolean assignVehicle(Vehicle vehicle) {
        if (!isAvailable) return false;
        this.vehicle = vehicle;
        this.isAvailable = false;
        return true;
    }

    public synchronized void removeVehicle() {
        this.vehicle = null;
        this.isAvailable = true;
    }

    // Getters
    public boolean isAvailable() { return isAvailable; }
    public SpotType getSpotType() { return spotType; }
}
```

### ParkingFloor Class

```java
public class ParkingFloor {
    private int floorNumber;
    private Map<SpotType, List<ParkingSpot>> spots;
    private DisplayBoard displayBoard;

    public ParkingFloor(int floorNumber) {
        this.floorNumber = floorNumber;
        this.spots = new HashMap<>();
        this.displayBoard = new DisplayBoard();
        for (SpotType type : SpotType.values()) {
            spots.put(type, new ArrayList<>());
        }
    }

    public void addSpot(ParkingSpot spot) {
        spots.get(spot.getSpotType()).add(spot);
    }

    public ParkingSpot getAvailableSpot(VehicleType vehicleType) {
        SpotType requiredType = mapVehicleToSpot(vehicleType);
        List<ParkingSpot> spotList = spots.get(requiredType);

        for (ParkingSpot spot : spotList) {
            if (spot.isAvailable()) return spot;
        }
        return null; // No spot available
    }

    private SpotType mapVehicleToSpot(VehicleType vehicleType) {
        switch (vehicleType) {
            case BIKE:  return SpotType.SMALL;
            case CAR:   return SpotType.MEDIUM;
            case TRUCK: return SpotType.LARGE;
            default: throw new IllegalArgumentException("Unknown vehicle type");
        }
    }

    public void updateDisplayBoard() {
        for (SpotType type : SpotType.values()) {
            long count = spots.get(type).stream()
                              .filter(ParkingSpot::isAvailable)
                              .count();
            displayBoard.update(floorNumber, type, (int) count);
        }
    }
}
```

### ParkingLot (Singleton)

```java
public class ParkingLot {
    private static ParkingLot instance;
    private String name;
    private List<ParkingFloor> floors;
    private List<EntryPanel> entryPanels;
    private List<ExitPanel> exitPanels;

    private ParkingLot() {
        floors = new ArrayList<>();
        entryPanels = new ArrayList<>();
        exitPanels = new ArrayList<>();
    }

    public static synchronized ParkingLot getInstance() {
        if (instance == null) {
            instance = new ParkingLot();
        }
        return instance;
    }

    public synchronized Ticket assignSpot(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.getAvailableSpot(vehicle.getType());
            if (spot != null) {
                spot.assignVehicle(vehicle);
                floor.updateDisplayBoard();
                return new Ticket(vehicle, spot);
            }
        }
        throw new ParkingFullException("No spot available for " + vehicle.getType());
    }

    public Payment freeSpot(Ticket ticket) {
        ParkingSpot spot = ticket.getSpot();
        spot.removeVehicle();

        long duration = Duration.between(ticket.getEntryTime(), LocalDateTime.now()).toHours();
        double amount = ParkingRate.calculate(ticket.getVehicle().getType(), duration);

        ticket.setStatus(TicketStatus.PAID);
        return new Payment(amount);
    }

    public void addFloor(ParkingFloor floor) {
        floors.add(floor);
    }
}
```

### Ticket & Payment

```java
public class Ticket {
    private String ticketId;
    private LocalDateTime entryTime;
    private Vehicle vehicle;
    private ParkingSpot spot;
    private TicketStatus status;

    public Ticket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = UUID.randomUUID().toString();
        this.entryTime = LocalDateTime.now();
        this.vehicle = vehicle;
        this.spot = spot;
        this.status = TicketStatus.ACTIVE;
    }

    // Getters and setters
}

public class Payment {
    private String paymentId;
    private double amount;
    private PaymentMethod method;
    private LocalDateTime paymentTime;

    public Payment(double amount) {
        this.paymentId = UUID.randomUUID().toString();
        this.amount = amount;
        this.paymentTime = LocalDateTime.now();
    }
}
```

### Pricing Strategy (Strategy Pattern)

```java
public interface PricingStrategy {
    double calculate(long hours);
}

public class CarPricing implements PricingStrategy {
    public double calculate(long hours) {
        return hours <= 1 ? 30 : 30 + (hours - 1) * 20;
    }
}

public class BikePricing implements PricingStrategy {
    public double calculate(long hours) {
        return hours <= 1 ? 10 : 10 + (hours - 1) * 5;
    }
}

public class TruckPricing implements PricingStrategy {
    public double calculate(long hours) {
        return hours <= 1 ? 50 : 50 + (hours - 1) * 40;
    }
}

public class ParkingRate {
    private static Map<VehicleType, PricingStrategy> strategies = Map.of(
        VehicleType.CAR, new CarPricing(),
        VehicleType.BIKE, new BikePricing(),
        VehicleType.TRUCK, new TruckPricing()
    );

    public static double calculate(VehicleType type, long hours) {
        return strategies.get(type).calculate(hours);
    }
}
```

---

## 🧪 Usage Example

```java
ParkingLot lot = ParkingLot.getInstance();

// Setup
ParkingFloor floor1 = new ParkingFloor(1);
floor1.addSpot(new ParkingSpot("1-S1", SpotType.SMALL));
floor1.addSpot(new ParkingSpot("1-M1", SpotType.MEDIUM));
floor1.addSpot(new ParkingSpot("1-L1", SpotType.LARGE));
lot.addFloor(floor1);

// Vehicle entry
Vehicle car = new Car("KA-01-1234", "White");
Ticket ticket = lot.assignSpot(car);

// Vehicle exit
Payment payment = lot.freeSpot(ticket);
payment.process(PaymentMethod.UPI);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Singleton** | `ParkingLot` — only one instance |
| **Strategy** | `PricingStrategy` — different pricing per vehicle type |
| **Factory** | Can be used to create Vehicle objects |
| **Observer** | `DisplayBoard` updates when spots change |

---

## ⚠️ Edge Cases

- Parking lot is full → throw `ParkingFullException`
- Vehicle already parked → validate by license plate
- Lost ticket → charge maximum amount
- Concurrent entry at multiple panels → synchronized spot assignment
- Power failure → persist ticket data to DB

---

## 🔑 Key Takeaways

1. **Singleton** ensures one parking lot instance
2. **Strategy Pattern** makes pricing extensible
3. **Synchronization** handles concurrent access
4. Vehicle-to-Spot mapping keeps assignment clean
5. DisplayBoard uses **Observer** to auto-update
