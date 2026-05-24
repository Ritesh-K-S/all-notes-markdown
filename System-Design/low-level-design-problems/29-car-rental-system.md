# 29 - Car Rental System

## 📋 Problem Statement

Design a car rental system that:
- Manages a fleet of vehicles across multiple locations
- Handles reservations, pickups, returns, and billing
- Supports different vehicle types and insurance options

---

## 📌 Requirements

### Functional Requirements
1. **Search vehicles** by type, location, date range
2. **Reserve** a vehicle for a date range
3. **Pickup** vehicle — verify identity, record mileage
4. **Return** vehicle — calculate charges, inspect damage
5. Vehicle types: **Sedan, SUV, Hatchback, Luxury**
6. **Multiple locations** — pickup and drop-off at different branches
7. **Insurance options** — Basic, Premium, Full Coverage
8. **Billing** — base rate + mileage + insurance + late fees

### Non-Functional Requirements
- No double-booking of vehicles
- Support concurrent reservations
- Maintain vehicle service history

---

## 🧩 Core Entities

```
Vehicle, VehicleType, Location, Reservation, Customer,
Bill, Insurance, VehicleInventory, RentalService
```

---

## 🔧 Enums

```java
public enum VehicleType {
    HATCHBACK, SEDAN, SUV, LUXURY
}

public enum VehicleStatus {
    AVAILABLE, RESERVED, RENTED, UNDER_MAINTENANCE, OUT_OF_SERVICE
}

public enum ReservationStatus {
    CONFIRMED, PICKED_UP, COMPLETED, CANCELLED, NO_SHOW
}

public enum InsuranceType {
    NONE(0), BASIC(200), PREMIUM(500), FULL_COVERAGE(1000);

    private final double dailyRate;
    InsuranceType(double rate) { this.dailyRate = rate; }
    public double getDailyRate() { return dailyRate; }
}
```

---

## 💻 Code Implementation

### Vehicle Class

```java
public class Vehicle {
    private String vehicleId;
    private String licensePlate;
    private String model;
    private VehicleType type;
    private VehicleStatus status;
    private double dailyRate;
    private double mileage;
    private Location currentLocation;

    public Vehicle(String id, String plate, String model,
                   VehicleType type, double dailyRate, Location location) {
        this.vehicleId = id;
        this.licensePlate = plate;
        this.model = model;
        this.type = type;
        this.dailyRate = dailyRate;
        this.status = VehicleStatus.AVAILABLE;
        this.mileage = 0;
        this.currentLocation = location;
    }

    public synchronized boolean reserve() {
        if (status != VehicleStatus.AVAILABLE) return false;
        this.status = VehicleStatus.RESERVED;
        return true;
    }

    public void pickup() {
        if (status != VehicleStatus.RESERVED) throw new IllegalStateException("Vehicle not reserved");
        this.status = VehicleStatus.RENTED;
    }

    public void returnVehicle(Location returnLocation, double newMileage) {
        this.status = VehicleStatus.AVAILABLE;
        this.mileage = newMileage;
        this.currentLocation = returnLocation;
    }

    public void markAvailable() { this.status = VehicleStatus.AVAILABLE; }

    // Getters
    public VehicleType getType() { return type; }
    public VehicleStatus getStatus() { return status; }
    public double getDailyRate() { return dailyRate; }
    public Location getCurrentLocation() { return currentLocation; }
    public double getMileage() { return mileage; }
    public String getVehicleId() { return vehicleId; }
}
```

### Reservation Class

```java
public class Reservation {
    private String reservationId;
    private Customer customer;
    private Vehicle vehicle;
    private Location pickupLocation;
    private Location returnLocation;
    private LocalDate startDate;
    private LocalDate endDate;
    private InsuranceType insurance;
    private ReservationStatus status;
    private LocalDateTime createdAt;

    public Reservation(Customer customer, Vehicle vehicle,
                       Location pickupLocation, Location returnLocation,
                       LocalDate startDate, LocalDate endDate,
                       InsuranceType insurance) {
        this.reservationId = UUID.randomUUID().toString();
        this.customer = customer;
        this.vehicle = vehicle;
        this.pickupLocation = pickupLocation;
        this.returnLocation = returnLocation;
        this.startDate = startDate;
        this.endDate = endDate;
        this.insurance = insurance;
        this.status = ReservationStatus.CONFIRMED;
        this.createdAt = LocalDateTime.now();
    }

    public long getRentalDays() {
        return ChronoUnit.DAYS.between(startDate, endDate);
    }

    public void pickup() {
        this.status = ReservationStatus.PICKED_UP;
        vehicle.pickup();
    }

    public Bill complete(double returnMileage, LocalDate actualReturnDate) {
        this.status = ReservationStatus.COMPLETED;
        vehicle.returnVehicle(returnLocation, returnMileage);
        return generateBill(actualReturnDate);
    }

    public void cancel() {
        this.status = ReservationStatus.CANCELLED;
        vehicle.markAvailable();
    }

    private Bill generateBill(LocalDate actualReturnDate) {
        long rentalDays = getRentalDays();
        double baseCharge = rentalDays * vehicle.getDailyRate();
        double insuranceCharge = rentalDays * insurance.getDailyRate();

        // Late return penalty
        double lateCharge = 0;
        if (actualReturnDate.isAfter(endDate)) {
            long lateDays = ChronoUnit.DAYS.between(endDate, actualReturnDate);
            lateCharge = lateDays * vehicle.getDailyRate() * 1.5; // 1.5x rate
        }

        // Different drop-off location surcharge
        double dropOffCharge = 0;
        if (!pickupLocation.equals(returnLocation)) {
            dropOffCharge = 500; // flat fee
        }

        return new Bill(reservationId, baseCharge, insuranceCharge, lateCharge, dropOffCharge);
    }

    // Getters
    public String getReservationId() { return reservationId; }
    public Vehicle getVehicle() { return vehicle; }
    public ReservationStatus getStatus() { return status; }
    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate() { return endDate; }
}
```

### Bill Class

```java
public class Bill {
    private String billId;
    private String reservationId;
    private double baseCharge;
    private double insuranceCharge;
    private double lateCharge;
    private double dropOffCharge;
    private double totalAmount;

    public Bill(String reservationId, double base, double insurance,
                double late, double dropOff) {
        this.billId = UUID.randomUUID().toString();
        this.reservationId = reservationId;
        this.baseCharge = base;
        this.insuranceCharge = insurance;
        this.lateCharge = late;
        this.dropOffCharge = dropOff;
        this.totalAmount = base + insurance + late + dropOff;
    }

    public void printBill() {
        System.out.println("═══════════════════════════════");
        System.out.println("  RENTAL BILL");
        System.out.println("═══════════════════════════════");
        System.out.printf("  Base Charge    : ₹%.2f%n", baseCharge);
        System.out.printf("  Insurance      : ₹%.2f%n", insuranceCharge);
        System.out.printf("  Late Fee       : ₹%.2f%n", lateCharge);
        System.out.printf("  Drop-off Fee   : ₹%.2f%n", dropOffCharge);
        System.out.println("───────────────────────────────");
        System.out.printf("  TOTAL          : ₹%.2f%n", totalAmount);
        System.out.println("═══════════════════════════════");
    }

    public double getTotalAmount() { return totalAmount; }
}
```

### Rental Service (Controller)

```java
public class RentalService {
    private Map<String, Vehicle> vehicles;
    private Map<String, Reservation> reservations;
    private Map<String, Location> locations;

    public RentalService() {
        this.vehicles = new ConcurrentHashMap<>();
        this.reservations = new ConcurrentHashMap<>();
        this.locations = new HashMap<>();
    }

    public List<Vehicle> searchVehicles(VehicleType type, String locationId,
                                         LocalDate start, LocalDate end) {
        return vehicles.values().stream()
            .filter(v -> v.getType() == type)
            .filter(v -> v.getStatus() == VehicleStatus.AVAILABLE)
            .filter(v -> v.getCurrentLocation().getId().equals(locationId))
            .collect(Collectors.toList());
    }

    public Reservation reserve(Customer customer, String vehicleId,
                                String pickupLocId, String returnLocId,
                                LocalDate start, LocalDate end,
                                InsuranceType insurance) {
        Vehicle vehicle = vehicles.get(vehicleId);
        if (vehicle == null) throw new VehicleNotFoundException("Vehicle not found");
        if (!vehicle.reserve()) throw new VehicleNotAvailableException("Vehicle unavailable");

        Reservation reservation = new Reservation(
            customer, vehicle,
            locations.get(pickupLocId), locations.get(returnLocId),
            start, end, insurance
        );

        reservations.put(reservation.getReservationId(), reservation);
        return reservation;
    }

    public void pickupVehicle(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) throw new ReservationNotFoundException("Not found");
        reservation.pickup();
    }

    public Bill returnVehicle(String reservationId, double mileage) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) throw new ReservationNotFoundException("Not found");
        return reservation.complete(mileage, LocalDate.now());
    }

    public void cancelReservation(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null) throw new ReservationNotFoundException("Not found");
        reservation.cancel();
    }
}
```

---

## 🧪 Usage Example

```java
RentalService service = new RentalService();

// Setup
Location bangalore = new Location("L1", "Bangalore Airport");
Location mysore = new Location("L2", "Mysore Branch");
service.addLocation(bangalore);
service.addLocation(mysore);

service.addVehicle(new Vehicle("V1", "KA-01-1234", "Swift", VehicleType.HATCHBACK, 800, bangalore));
service.addVehicle(new Vehicle("V2", "KA-01-5678", "Creta", VehicleType.SUV, 2000, bangalore));

Customer customer = new Customer("C1", "Ritesh", "DL-12345678");

// Search
List<Vehicle> available = service.searchVehicles(VehicleType.SUV, "L1",
    LocalDate.of(2026, 5, 1), LocalDate.of(2026, 5, 5));

// Reserve
Reservation res = service.reserve(customer, "V2", "L1", "L2",
    LocalDate.of(2026, 5, 1), LocalDate.of(2026, 5, 5), InsuranceType.PREMIUM);

// Pickup
service.pickupVehicle(res.getReservationId());

// Return (different location)
Bill bill = service.returnVehicle(res.getReservationId(), 500);
bill.printBill();
// Base: 4 days × ₹2000 = ₹8000
// Insurance: 4 days × ₹500 = ₹2000
// Drop-off fee: ₹500
// Total: ₹10500
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Insurance types, Pricing strategies |
| **Factory** | Vehicle creation by type |
| **State** | Vehicle status transitions |
| **Observer** | Reservation notifications |

---

## 🔑 Key Takeaways

1. **Vehicle.reserve()** is synchronized — prevents double-booking
2. **Bill calculation** includes base + insurance + late fees + location surcharge
3. **Multi-location** support with drop-off fee for different return location
4. Vehicle status follows **state machine**: AVAILABLE → RESERVED → RENTED → AVAILABLE
5. Late return penalty at **1.5x daily rate**
