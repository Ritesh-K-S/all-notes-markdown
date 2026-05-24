# 31 - Airline / Flight Booking System

## 📋 Problem Statement

Design an airline booking system that:
- Allows searching for flights between cities
- Supports seat selection across classes (Economy, Business, First)
- Manages booking with PNR, payment, and check-in

---

## 📌 Requirements

### Functional Requirements
1. **Search flights** by source, destination, date
2. **View seat map** and select seats
3. **Book tickets** — passenger details, fare, PNR generation
4. **Multiple passengers** per booking (PNR)
5. **Seat classes**: Economy, Business, First
6. **Check-in** — boarding pass generation
7. **Cancel booking** with refund policy
8. **Flight status** — On Time, Delayed, Cancelled

### Non-Functional Requirements
- No double-booking of seats
- Handle concurrent seat selections
- PNR lookup in O(1)

---

## 🧩 Core Entities

```
Flight, Aircraft, Seat, SeatClass, Booking, Passenger,
BoardingPass, Airport, FlightSearch, BookingService
```

---

## 🔧 Enums

```java
public enum SeatClass {
    ECONOMY, PREMIUM_ECONOMY, BUSINESS, FIRST
}

public enum SeatStatus {
    AVAILABLE, LOCKED, BOOKED
}

public enum BookingStatus {
    CONFIRMED, CANCELLED, CHECKED_IN, COMPLETED
}

public enum FlightStatus {
    SCHEDULED, BOARDING, DEPARTED, ARRIVED, DELAYED, CANCELLED
}
```

---

## 💻 Code Implementation

### Airport & Flight

```java
public class Airport {
    private String code; // IATA code: BLR, DEL, BOM
    private String name;
    private String city;

    public Airport(String code, String name, String city) {
        this.code = code;
        this.name = name;
        this.city = city;
    }

    public String getCode() { return code; }
    public String getCity() { return city; }
}

public class Flight {
    private String flightNumber; // AI-202
    private Airport source;
    private Airport destination;
    private LocalDateTime departure;
    private LocalDateTime arrival;
    private Aircraft aircraft;
    private FlightStatus status;
    private Map<SeatClass, Double> baseFares; // class → price

    public Flight(String flightNumber, Airport source, Airport destination,
                  LocalDateTime departure, LocalDateTime arrival, Aircraft aircraft) {
        this.flightNumber = flightNumber;
        this.source = source;
        this.destination = destination;
        this.departure = departure;
        this.arrival = arrival;
        this.aircraft = aircraft;
        this.status = FlightStatus.SCHEDULED;
        this.baseFares = new EnumMap<>(SeatClass.class);
    }

    public void setFare(SeatClass seatClass, double fare) {
        baseFares.put(seatClass, fare);
    }

    public double getFare(SeatClass seatClass) {
        return baseFares.getOrDefault(seatClass, 0.0);
    }

    public List<Seat> getAvailableSeats(SeatClass seatClass) {
        return aircraft.getSeats().stream()
            .filter(s -> s.getSeatClass() == seatClass)
            .filter(s -> s.getStatus() == SeatStatus.AVAILABLE)
            .collect(Collectors.toList());
    }

    public String getDuration() {
        Duration d = Duration.between(departure, arrival);
        return d.toHours() + "h " + (d.toMinutes() % 60) + "m";
    }

    // Getters
    public String getFlightNumber() { return flightNumber; }
    public Airport getSource() { return source; }
    public Airport getDestination() { return destination; }
    public LocalDateTime getDeparture() { return departure; }
    public FlightStatus getStatus() { return status; }
    public Aircraft getAircraft() { return aircraft; }
}
```

### Aircraft & Seat

```java
public class Aircraft {
    private String aircraftId;
    private String model; // Boeing 737, Airbus A320
    private List<Seat> seats;

    public Aircraft(String id, String model) {
        this.aircraftId = id;
        this.model = model;
        this.seats = new ArrayList<>();
    }

    public void addSeat(Seat seat) { seats.add(seat); }

    public List<Seat> getSeats() { return seats; }

    public Seat getSeat(String seatNumber) {
        return seats.stream()
            .filter(s -> s.getSeatNumber().equals(seatNumber))
            .findFirst()
            .orElseThrow(() -> new SeatNotFoundException("Seat " + seatNumber + " not found"));
    }
}

public class Seat {
    private String seatNumber; // 12A, 15C
    private SeatClass seatClass;
    private SeatStatus status;
    private boolean isWindow;
    private boolean isAisle;

    public Seat(String seatNumber, SeatClass seatClass, boolean isWindow, boolean isAisle) {
        this.seatNumber = seatNumber;
        this.seatClass = seatClass;
        this.isWindow = isWindow;
        this.isAisle = isAisle;
        this.status = SeatStatus.AVAILABLE;
    }

    public synchronized boolean lock() {
        if (status != SeatStatus.AVAILABLE) return false;
        status = SeatStatus.LOCKED;
        return true;
    }

    public synchronized boolean book() {
        if (status != SeatStatus.LOCKED) return false;
        status = SeatStatus.BOOKED;
        return true;
    }

    public void release() {
        status = SeatStatus.AVAILABLE;
    }

    // Getters
    public String getSeatNumber() { return seatNumber; }
    public SeatClass getSeatClass() { return seatClass; }
    public SeatStatus getStatus() { return status; }
}
```

### Passenger & Booking

```java
public class Passenger {
    private String name;
    private int age;
    private String passportNumber;
    private Seat assignedSeat;

    public Passenger(String name, int age, String passportNumber) {
        this.name = name;
        this.age = age;
        this.passportNumber = passportNumber;
    }

    public void assignSeat(Seat seat) { this.assignedSeat = seat; }
    public String getName() { return name; }
    public Seat getAssignedSeat() { return assignedSeat; }
}

public class Booking {
    private String pnr; // 6-char alphanumeric
    private Flight flight;
    private List<Passenger> passengers;
    private BookingStatus status;
    private double totalFare;
    private LocalDateTime bookedAt;
    private String contactEmail;
    private String contactPhone;

    public Booking(Flight flight, List<Passenger> passengers, double totalFare) {
        this.pnr = generatePNR();
        this.flight = flight;
        this.passengers = passengers;
        this.totalFare = totalFare;
        this.status = BookingStatus.CONFIRMED;
        this.bookedAt = LocalDateTime.now();
    }

    private String generatePNR() {
        String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
        StringBuilder pnr = new StringBuilder();
        Random random = new SecureRandom();
        for (int i = 0; i < 6; i++) {
            pnr.append(chars.charAt(random.nextInt(chars.length())));
        }
        return pnr.toString();
    }

    public void cancel() {
        this.status = BookingStatus.CANCELLED;
        for (Passenger p : passengers) {
            if (p.getAssignedSeat() != null) {
                p.getAssignedSeat().release();
            }
        }
    }

    public void checkIn() {
        if (status != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot check in — status: " + status);
        }
        this.status = BookingStatus.CHECKED_IN;
    }

    public List<BoardingPass> generateBoardingPasses() {
        if (status != BookingStatus.CHECKED_IN) {
            throw new IllegalStateException("Must check in first");
        }
        List<BoardingPass> passes = new ArrayList<>();
        for (Passenger p : passengers) {
            passes.add(new BoardingPass(pnr, flight, p));
        }
        return passes;
    }

    // Getters
    public String getPnr() { return pnr; }
    public BookingStatus getStatus() { return status; }
    public double getTotalFare() { return totalFare; }
    public List<Passenger> getPassengers() { return passengers; }
    public Flight getFlight() { return flight; }
}
```

### Boarding Pass

```java
public class BoardingPass {
    private String pnr;
    private String flightNumber;
    private String passengerName;
    private String seatNumber;
    private String source;
    private String destination;
    private LocalDateTime departure;
    private String gate;
    private String boardingGroup;

    public BoardingPass(String pnr, Flight flight, Passenger passenger) {
        this.pnr = pnr;
        this.flightNumber = flight.getFlightNumber();
        this.passengerName = passenger.getName();
        this.seatNumber = passenger.getAssignedSeat().getSeatNumber();
        this.source = flight.getSource().getCode();
        this.destination = flight.getDestination().getCode();
        this.departure = flight.getDeparture();
    }

    public void print() {
        System.out.println("╔═══════════════════════════════════════╗");
        System.out.println("║         ✈ BOARDING PASS               ║");
        System.out.println("╠═══════════════════════════════════════╣");
        System.out.printf("║  Name    : %-26s ║%n", passengerName);
        System.out.printf("║  Flight  : %-26s ║%n", flightNumber);
        System.out.printf("║  Route   : %-6s → %-17s ║%n", source, destination);
        System.out.printf("║  Seat    : %-26s ║%n", seatNumber);
        System.out.printf("║  PNR     : %-26s ║%n", pnr);
        System.out.println("╚═══════════════════════════════════════╝");
    }
}
```

### Booking Service (Controller)

```java
public class BookingService {
    private Map<String, Flight> flights; // flightNumber → Flight
    private Map<String, Booking> bookings; // PNR → Booking
    private Map<String, Airport> airports;

    public BookingService() {
        this.flights = new ConcurrentHashMap<>();
        this.bookings = new ConcurrentHashMap<>();
        this.airports = new HashMap<>();
    }

    public List<Flight> searchFlights(String sourceCode, String destCode, LocalDate date) {
        return flights.values().stream()
            .filter(f -> f.getSource().getCode().equals(sourceCode))
            .filter(f -> f.getDestination().getCode().equals(destCode))
            .filter(f -> f.getDeparture().toLocalDate().equals(date))
            .filter(f -> f.getStatus() == FlightStatus.SCHEDULED)
            .sorted(Comparator.comparing(Flight::getDeparture))
            .collect(Collectors.toList());
    }

    public Booking bookFlight(String flightNumber, List<Passenger> passengers,
                               List<String> seatNumbers) {
        Flight flight = flights.get(flightNumber);
        if (flight == null) throw new FlightNotFoundException("Flight not found");
        if (passengers.size() != seatNumbers.size()) {
            throw new IllegalArgumentException("Passenger count must match seat count");
        }

        // Step 1: Lock all requested seats
        List<Seat> lockedSeats = new ArrayList<>();
        try {
            for (String seatNum : seatNumbers) {
                Seat seat = flight.getAircraft().getSeat(seatNum);
                if (!seat.lock()) {
                    throw new SeatNotAvailableException("Seat " + seatNum + " not available");
                }
                lockedSeats.add(seat);
            }

            // Step 2: Assign seats and calculate fare
            double totalFare = 0;
            for (int i = 0; i < passengers.size(); i++) {
                Passenger p = passengers.get(i);
                Seat seat = lockedSeats.get(i);
                p.assignSeat(seat);
                totalFare += flight.getFare(seat.getSeatClass());
            }

            // Step 3: Confirm seats
            for (Seat seat : lockedSeats) {
                seat.book();
            }

            // Step 4: Create booking
            Booking booking = new Booking(flight, passengers, totalFare);
            bookings.put(booking.getPnr(), booking);

            System.out.println("Booking confirmed! PNR: " + booking.getPnr());
            return booking;

        } catch (Exception e) {
            // Rollback: release locked seats
            for (Seat seat : lockedSeats) {
                seat.release();
            }
            throw e;
        }
    }

    public void cancelBooking(String pnr) {
        Booking booking = bookings.get(pnr);
        if (booking == null) throw new BookingNotFoundException("PNR not found");

        double refund = calculateRefund(booking);
        booking.cancel();

        System.out.printf("Booking %s cancelled. Refund: ₹%.2f%n", pnr, refund);
    }

    private double calculateRefund(Booking booking) {
        long hoursToFlight = Duration.between(
            LocalDateTime.now(), booking.getFlight().getDeparture()).toHours();

        if (hoursToFlight > 48) return booking.getTotalFare() * 0.90; // 90% refund
        if (hoursToFlight > 24) return booking.getTotalFare() * 0.50; // 50% refund
        if (hoursToFlight > 4)  return booking.getTotalFare() * 0.25; // 25% refund
        return 0; // no refund
    }

    public void checkIn(String pnr) {
        Booking booking = bookings.get(pnr);
        if (booking == null) throw new BookingNotFoundException("PNR not found");
        booking.checkIn();

        List<BoardingPass> passes = booking.generateBoardingPasses();
        passes.forEach(BoardingPass::print);
    }

    public Booking lookupBooking(String pnr) {
        Booking booking = bookings.get(pnr);
        if (booking == null) throw new BookingNotFoundException("PNR: " + pnr + " not found");
        return booking;
    }

    public void addFlight(Flight flight) { flights.put(flight.getFlightNumber(), flight); }
    public void addAirport(Airport airport) { airports.put(airport.getCode(), airport); }
}
```

---

## 🧪 Usage Example

```java
BookingService service = new BookingService();

// Setup airports
Airport blr = new Airport("BLR", "Kempegowda International", "Bangalore");
Airport del = new Airport("DEL", "Indira Gandhi International", "Delhi");

// Setup aircraft with seats
Aircraft boeing = new Aircraft("AC1", "Boeing 737");
for (int row = 1; row <= 5; row++) {  // First class: rows 1-5
    boeing.addSeat(new Seat(row + "A", SeatClass.FIRST, true, false));
    boeing.addSeat(new Seat(row + "B", SeatClass.FIRST, false, true));
}
for (int row = 6; row <= 30; row++) { // Economy: rows 6-30
    boeing.addSeat(new Seat(row + "A", SeatClass.ECONOMY, true, false));
    boeing.addSeat(new Seat(row + "B", SeatClass.ECONOMY, false, false));
    boeing.addSeat(new Seat(row + "C", SeatClass.ECONOMY, false, true));
}

// Create flight
Flight flight = new Flight("AI-202", blr, del,
    LocalDateTime.of(2026, 5, 15, 6, 0),
    LocalDateTime.of(2026, 5, 15, 8, 30), boeing);
flight.setFare(SeatClass.ECONOMY, 5000);
flight.setFare(SeatClass.FIRST, 15000);
service.addFlight(flight);

// Search
List<Flight> results = service.searchFlights("BLR", "DEL", LocalDate.of(2026, 5, 15));

// Book
List<Passenger> passengers = List.of(
    new Passenger("Ritesh Singh", 28, "P1234567"),
    new Passenger("Amit Kumar", 30, "P7654321")
);
Booking booking = service.bookFlight("AI-202", passengers, List.of("12A", "12B"));
// → PNR: A3X7K9

// Check-in
service.checkIn(booking.getPnr());
// → Prints boarding passes for both passengers

// Cancel
service.cancelBooking(booking.getPnr());
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Refund policy (time-based), fare calculation |
| **State** | Seat status (AVAILABLE → LOCKED → BOOKED), Booking status |
| **Factory** | Aircraft seat configuration |
| **Builder** | Flight with optional fares, boarding pass |

---

## ⚠️ Edge Cases

- Concurrent seat booking → `synchronized lock()` prevents double-booking
- Partial seat lock failure → rollback all locked seats
- Cancel close to departure → tiered refund policy
- Flight cancelled by airline → full refund to all passengers
- Round-trip booking → two separate bookings linked
- Infant/child passenger → special fare rules

---

## 🔑 Key Takeaways

1. **Two-phase seat booking**: LOCK → BOOK with rollback on failure
2. **PNR** is a 6-character unique booking reference (O(1) lookup via HashMap)
3. **Tiered refund policy** — >48h: 90%, >24h: 50%, >4h: 25%, <4h: 0%
4. **Boarding pass** generated only after check-in
5. **Seat.lock()** is synchronized to prevent race conditions
