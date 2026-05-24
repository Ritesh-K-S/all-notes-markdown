# 15 - Cab Booking System (Uber/Ola)

## 📋 Problem Statement

Design a cab booking system that:
- Matches riders with nearby drivers
- Supports ride booking, tracking, and payment
- Handles different vehicle and ride types

---

## 📌 Requirements

### Functional Requirements
1. **Rider requests** a ride (pickup, drop, ride type)
2. **Find nearby drivers** within radius
3. **Driver accepts/rejects** ride request
4. **Track ride** in real-time (simulated)
5. **Fare estimation** before booking
6. **Payment** after ride completion
7. **Rating** — rider rates driver and vice versa
8. Ride types: **Mini, Sedan, SUV, Premium**
9. **Surge pricing** during peak hours

### Non-Functional Requirements
- Efficient driver matching
- Handle concurrent ride requests
- Real-time location updates

---

## 🧩 Core Entities

```
Rider, Driver, Ride, Vehicle, Location, Payment,
RideRequest, FareCalculator, DriverMatchingStrategy, Rating
```

---

## 📐 Class Diagram

```
┌──────────────────┐     ┌──────────────────┐
│     Rider         │     │     Driver        │
│  - riderId        │     │  - driverId       │
│  - name           │     │  - name           │
│  - location       │     │  - location       │
│  - rating         │     │  - vehicle        │
│  + requestRide()  │     │  - status         │
│  + cancelRide()   │     │  - rating         │
└──────────────────┘     │  + acceptRide()   │
                          │  + rejectRide()   │
                          │  + updateLoc()    │
                          └──────────────────┘

┌──────────────────────────────────────────────────┐
│                      Ride                         │
│  - rideId: String                                 │
│  - rider: Rider                                   │
│  - driver: Driver                                 │
│  - pickup: Location                               │
│  - drop: Location                                 │
│  - status: RideStatus                             │
│  - rideType: RideType                             │
│  - fare: double                                   │
│  - startTime / endTime                            │
│  - distance: double                               │
└──────────────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum RideStatus {
    REQUESTED, ACCEPTED, DRIVER_ARRIVING, IN_PROGRESS, COMPLETED, CANCELLED
}

public enum RideType {
    MINI, SEDAN, SUV, PREMIUM
}

public enum DriverStatus {
    AVAILABLE, ON_RIDE, OFFLINE
}
```

---

## 💻 Code Implementation

### Location Class

```java
public class Location {
    private double latitude;
    private double longitude;

    public Location(double latitude, double longitude) {
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double distanceTo(Location other) {
        // Haversine formula simplified
        double dx = this.latitude - other.latitude;
        double dy = this.longitude - other.longitude;
        return Math.sqrt(dx * dx + dy * dy) * 111; // approx km
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }
}
```

### Driver & Rider

```java
public class Driver {
    private String driverId;
    private String name;
    private Location location;
    private Vehicle vehicle;
    private DriverStatus status;
    private double rating;
    private int totalRides;

    public Driver(String driverId, String name, Vehicle vehicle) {
        this.driverId = driverId;
        this.name = name;
        this.vehicle = vehicle;
        this.status = DriverStatus.AVAILABLE;
        this.rating = 5.0;
        this.totalRides = 0;
    }

    public boolean acceptRide() {
        if (status != DriverStatus.AVAILABLE) return false;
        this.status = DriverStatus.ON_RIDE;
        return true;
    }

    public void completeRide() {
        this.status = DriverStatus.AVAILABLE;
        this.totalRides++;
    }

    public void updateRating(double newRating) {
        this.rating = ((rating * totalRides) + newRating) / (totalRides + 1);
    }

    public void updateLocation(Location location) { this.location = location; }
    public Location getLocation() { return location; }
    public DriverStatus getStatus() { return status; }
    public Vehicle getVehicle() { return vehicle; }
    public String getDriverId() { return driverId; }
}

public class Rider {
    private String riderId;
    private String name;
    private Location location;
    private double rating;

    public Rider(String riderId, String name) {
        this.riderId = riderId;
        this.name = name;
        this.rating = 5.0;
    }

    public Location getLocation() { return location; }
    public void setLocation(Location location) { this.location = location; }
}
```

### Driver Matching Strategy

```java
public interface DriverMatchingStrategy {
    Driver findDriver(List<Driver> drivers, Location pickup, RideType rideType);
}

// Nearest available driver
public class NearestDriverStrategy implements DriverMatchingStrategy {
    @Override
    public Driver findDriver(List<Driver> drivers, Location pickup, RideType rideType) {
        return drivers.stream()
            .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
            .filter(d -> d.getVehicle().supportsRideType(rideType))
            .min(Comparator.comparingDouble(d -> d.getLocation().distanceTo(pickup)))
            .orElse(null);
    }
}

// Highest rated driver within radius
public class HighestRatedStrategy implements DriverMatchingStrategy {
    private static final double MAX_RADIUS_KM = 5.0;

    @Override
    public Driver findDriver(List<Driver> drivers, Location pickup, RideType rideType) {
        return drivers.stream()
            .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
            .filter(d -> d.getVehicle().supportsRideType(rideType))
            .filter(d -> d.getLocation().distanceTo(pickup) <= MAX_RADIUS_KM)
            .max(Comparator.comparingDouble(Driver::getRating))
            .orElse(null);
    }
}
```

### Fare Calculator (Strategy Pattern)

```java
public interface FareCalculator {
    double calculateFare(double distanceKm, RideType rideType, double surgeMultiplier);
}

public class DefaultFareCalculator implements FareCalculator {
    private static final Map<RideType, Double> BASE_FARE = Map.of(
        RideType.MINI, 30.0,
        RideType.SEDAN, 50.0,
        RideType.SUV, 80.0,
        RideType.PREMIUM, 120.0
    );

    private static final Map<RideType, Double> PER_KM_RATE = Map.of(
        RideType.MINI, 8.0,
        RideType.SEDAN, 12.0,
        RideType.SUV, 16.0,
        RideType.PREMIUM, 25.0
    );

    @Override
    public double calculateFare(double distanceKm, RideType rideType, double surgeMultiplier) {
        double baseFare = BASE_FARE.get(rideType);
        double distanceFare = PER_KM_RATE.get(rideType) * distanceKm;
        return (baseFare + distanceFare) * surgeMultiplier;
    }
}
```

### Ride & RideService

```java
public class Ride {
    private String rideId;
    private Rider rider;
    private Driver driver;
    private Location pickup;
    private Location drop;
    private RideType rideType;
    private RideStatus status;
    private double fare;
    private double distance;
    private LocalDateTime startTime;
    private LocalDateTime endTime;

    public Ride(Rider rider, Location pickup, Location drop, RideType rideType) {
        this.rideId = UUID.randomUUID().toString();
        this.rider = rider;
        this.pickup = pickup;
        this.drop = drop;
        this.rideType = rideType;
        this.status = RideStatus.REQUESTED;
        this.distance = pickup.distanceTo(drop);
    }

    public void assignDriver(Driver driver) {
        this.driver = driver;
        this.status = RideStatus.ACCEPTED;
    }

    public void start() {
        this.status = RideStatus.IN_PROGRESS;
        this.startTime = LocalDateTime.now();
    }

    public void complete(double fare) {
        this.status = RideStatus.COMPLETED;
        this.endTime = LocalDateTime.now();
        this.fare = fare;
    }

    public void cancel() {
        this.status = RideStatus.CANCELLED;
    }

    // Getters
    public double getDistance() { return distance; }
    public RideType getRideType() { return rideType; }
    public Driver getDriver() { return driver; }
    public RideStatus getStatus() { return status; }
}

public class RideService {
    private List<Driver> allDrivers;
    private Map<String, Ride> rides;
    private DriverMatchingStrategy matchingStrategy;
    private FareCalculator fareCalculator;
    private double surgeMultiplier;

    public RideService(List<Driver> drivers) {
        this.allDrivers = drivers;
        this.rides = new ConcurrentHashMap<>();
        this.matchingStrategy = new NearestDriverStrategy();
        this.fareCalculator = new DefaultFareCalculator();
        this.surgeMultiplier = 1.0;
    }

    public double estimateFare(Location pickup, Location drop, RideType rideType) {
        double distance = pickup.distanceTo(drop);
        return fareCalculator.calculateFare(distance, rideType, surgeMultiplier);
    }

    public Ride requestRide(Rider rider, Location pickup, Location drop, RideType rideType) {
        Ride ride = new Ride(rider, pickup, drop, rideType);

        Driver driver = matchingStrategy.findDriver(allDrivers, pickup, rideType);
        if (driver == null) throw new NoDriverAvailableException("No drivers available nearby");

        driver.acceptRide();
        ride.assignDriver(driver);
        rides.put(ride.getRideId(), ride);

        return ride;
    }

    public void startRide(String rideId) {
        Ride ride = rides.get(rideId);
        ride.start();
    }

    public void completeRide(String rideId) {
        Ride ride = rides.get(rideId);
        double fare = fareCalculator.calculateFare(ride.getDistance(), ride.getRideType(), surgeMultiplier);
        ride.complete(fare);
        ride.getDriver().completeRide();
    }

    public void setSurgeMultiplier(double multiplier) {
        this.surgeMultiplier = multiplier;
    }
}
```

---

## 🧪 Usage Example

```java
List<Driver> drivers = List.of(
    new Driver("D1", "Raju", new Vehicle("KA-01-1234", RideType.SEDAN)),
    new Driver("D2", "Suresh", new Vehicle("KA-01-5678", RideType.MINI))
);
drivers.get(0).updateLocation(new Location(12.97, 77.59));
drivers.get(1).updateLocation(new Location(12.98, 77.60));

RideService service = new RideService(drivers);

Rider rider = new Rider("R1", "Ritesh");
Location pickup = new Location(12.97, 77.58);
Location drop = new Location(12.95, 77.55);

// Estimate fare
double estimate = service.estimateFare(pickup, drop, RideType.SEDAN);
System.out.println("Estimated fare: ₹" + estimate);

// Request ride
Ride ride = service.requestRide(rider, pickup, drop, RideType.SEDAN);

// Start and complete
service.startRide(ride.getRideId());
service.completeRide(ride.getRideId());
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Driver matching, Fare calculation |
| **Observer** | Ride status notifications |
| **State** | Ride status transitions |
| **Factory** | Creating rides based on type |

---

## 🔑 Key Takeaways

1. **Strategy Pattern** for driver matching — nearest, highest rated, etc.
2. **Fare calculator** is pluggable — supports surge pricing
3. **Location-based matching** with distance calculation
4. Ride follows a **state machine**: REQUESTED → ACCEPTED → IN_PROGRESS → COMPLETED
5. **Rating system** uses running average formula
