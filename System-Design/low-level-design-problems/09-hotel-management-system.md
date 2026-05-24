# 09 - Hotel Management System

## 📋 Problem Statement

Design a hotel management system that:
- Manages room bookings and availability
- Supports different room types and pricing
- Handles check-in, check-out, and billing

---

## 📌 Requirements

### Functional Requirements
1. **Search rooms** by type, date range, and availability
2. **Book a room** for a date range
3. **Check-in / Check-out** guests
4. Support room types: **Standard, Deluxe, Suite**
5. **Housekeeping** management for rooms
6. **Billing** — room charges + services (food, laundry, etc.)
7. **Cancel booking** with refund policy
8. **Room key card** management

### Non-Functional Requirements
- Handle concurrent booking of same room
- Support multiple hotels/branches
- Notification for booking confirmation

---

## 🧩 Core Entities

```
Hotel, Room, RoomType, Guest, Booking, Payment, 
HouseKeeper, Service, Bill, RoomKeyCard
```

---

## 📐 Class Diagram

```
┌──────────────┐     ┌──────────────────┐
│    Hotel      │────▶│      Room         │
│  - name       │1  * │  - roomNumber     │
│  - address    │     │  - type: RoomType │
│  - rooms[]    │     │  - status         │
│  - bookings[] │     │  - pricePerNight  │
│               │     │  - floor          │
└──────────────┘     └──────────────────┘

┌──────────────────────────────────────────────────┐
│                    Booking                        │
│  - bookingId: String                              │
│  - guest: Guest                                   │
│  - room: Room                                     │
│  - checkInDate: LocalDate                         │
│  - checkOutDate: LocalDate                        │
│  - status: BookingStatus                          │
│  - services: List<Service>                        │
│  + calculateTotal(): double                       │
│  + addService(service): void                      │
└──────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│    Guest          │    │   Service         │
│  - name           │    │  - name           │
│  - phone          │    │  - price          │
│  - email          │    │  - type           │
│  - idProof        │    │  (Food, Laundry,  │
│                   │    │   Spa, Minibar)   │
└──────────────────┘    └──────────────────┘
```

---

## 🔧 Enums

```java
public enum RoomType {
    STANDARD, DELUXE, SUITE, PRESIDENTIAL
}

public enum RoomStatus {
    AVAILABLE, BOOKED, OCCUPIED, UNDER_MAINTENANCE, CLEANING
}

public enum BookingStatus {
    CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED, NO_SHOW
}

public enum ServiceType {
    FOOD, LAUNDRY, SPA, MINIBAR, ROOM_SERVICE, PARKING
}
```

---

## 💻 Code Implementation

### Room Class

```java
public class Room {
    private String roomNumber;
    private RoomType type;
    private RoomStatus status;
    private double pricePerNight;
    private int floor;

    public Room(String roomNumber, RoomType type, double pricePerNight, int floor) {
        this.roomNumber = roomNumber;
        this.type = type;
        this.pricePerNight = pricePerNight;
        this.floor = floor;
        this.status = RoomStatus.AVAILABLE;
    }

    public synchronized boolean book() {
        if (status != RoomStatus.AVAILABLE) return false;
        this.status = RoomStatus.BOOKED;
        return true;
    }

    public void checkIn() {
        if (status != RoomStatus.BOOKED) throw new IllegalStateException("Room not booked");
        this.status = RoomStatus.OCCUPIED;
    }

    public void checkOut() {
        this.status = RoomStatus.CLEANING;
    }

    public void markAvailable() {
        this.status = RoomStatus.AVAILABLE;
    }

    // Getters
    public RoomType getType() { return type; }
    public RoomStatus getStatus() { return status; }
    public double getPricePerNight() { return pricePerNight; }
    public String getRoomNumber() { return roomNumber; }
}
```

### Booking Class

```java
public class Booking {
    private String bookingId;
    private Guest guest;
    private Room room;
    private LocalDate checkInDate;
    private LocalDate checkOutDate;
    private BookingStatus status;
    private List<Service> services;
    private LocalDateTime createdAt;

    public Booking(Guest guest, Room room, LocalDate checkIn, LocalDate checkOut) {
        this.bookingId = UUID.randomUUID().toString();
        this.guest = guest;
        this.room = room;
        this.checkInDate = checkIn;
        this.checkOutDate = checkOut;
        this.status = BookingStatus.CONFIRMED;
        this.services = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
    }

    public void addService(Service service) {
        services.add(service);
    }

    public double calculateTotal() {
        long nights = ChronoUnit.DAYS.between(checkInDate, checkOutDate);
        double roomCharge = nights * room.getPricePerNight();
        double serviceCharge = services.stream().mapToDouble(Service::getPrice).sum();
        return roomCharge + serviceCharge;
    }

    public void checkIn() {
        room.checkIn();
        this.status = BookingStatus.CHECKED_IN;
    }

    public Bill checkOut() {
        room.checkOut();
        this.status = BookingStatus.CHECKED_OUT;
        return new Bill(this, calculateTotal());
    }

    public void cancel() {
        room.markAvailable();
        this.status = BookingStatus.CANCELLED;
    }

    // Getters
    public String getBookingId() { return bookingId; }
    public BookingStatus getStatus() { return status; }
    public Room getRoom() { return room; }
}
```

### Hotel Service (Controller)

```java
public class HotelService {
    private Hotel hotel;
    private Map<String, Booking> bookings;
    private NotificationService notificationService;

    public HotelService(Hotel hotel) {
        this.hotel = hotel;
        this.bookings = new ConcurrentHashMap<>();
        this.notificationService = new NotificationService();
    }

    public List<Room> searchAvailableRooms(RoomType type, LocalDate checkIn, LocalDate checkOut) {
        return hotel.getRooms().stream()
            .filter(r -> r.getType() == type)
            .filter(r -> r.getStatus() == RoomStatus.AVAILABLE)
            .filter(r -> isRoomAvailable(r, checkIn, checkOut))
            .collect(Collectors.toList());
    }

    private boolean isRoomAvailable(Room room, LocalDate checkIn, LocalDate checkOut) {
        return bookings.values().stream()
            .filter(b -> b.getRoom().equals(room))
            .filter(b -> b.getStatus() == BookingStatus.CONFIRMED || b.getStatus() == BookingStatus.CHECKED_IN)
            .noneMatch(b -> datesOverlap(b.getCheckInDate(), b.getCheckOutDate(), checkIn, checkOut));
    }

    private boolean datesOverlap(LocalDate s1, LocalDate e1, LocalDate s2, LocalDate e2) {
        return s1.isBefore(e2) && s2.isBefore(e1);
    }

    public Booking bookRoom(Guest guest, Room room, LocalDate checkIn, LocalDate checkOut) {
        if (!room.book()) {
            throw new RoomNotAvailableException("Room not available");
        }

        Booking booking = new Booking(guest, room, checkIn, checkOut);
        bookings.put(booking.getBookingId(), booking);

        notificationService.sendBookingConfirmation(guest, booking);
        return booking;
    }

    public void checkIn(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) throw new BookingNotFoundException("Booking not found");
        booking.checkIn();

        // Issue key card
        RoomKeyCard keyCard = new RoomKeyCard(booking.getRoom(), booking);
        System.out.println("Key card issued for room " + booking.getRoom().getRoomNumber());
    }

    public Bill checkOut(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) throw new BookingNotFoundException("Booking not found");

        Bill bill = booking.checkOut();
        System.out.println("Total Bill: ₹" + bill.getAmount());
        return bill;
    }

    public void cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) throw new BookingNotFoundException("Booking not found");

        booking.cancel();
        // Refund based on cancellation policy
        double refund = calculateRefund(booking);
        System.out.println("Booking cancelled. Refund: ₹" + refund);
    }

    private double calculateRefund(Booking booking) {
        long daysUntilCheckIn = ChronoUnit.DAYS.between(LocalDate.now(), booking.getCheckInDate());
        if (daysUntilCheckIn > 7) return booking.calculateTotal(); // Full refund
        if (daysUntilCheckIn > 3) return booking.calculateTotal() * 0.5; // 50% refund
        return 0; // No refund
    }
}
```

---

## 🧪 Usage Example

```java
Hotel hotel = new Hotel("Grand Palace", "Mumbai");
hotel.addRoom(new Room("101", RoomType.STANDARD, 3000, 1));
hotel.addRoom(new Room("201", RoomType.DELUXE, 5000, 2));
hotel.addRoom(new Room("301", RoomType.SUITE, 10000, 3));

HotelService service = new HotelService(hotel);

// Search
List<Room> rooms = service.searchAvailableRooms(RoomType.DELUXE, 
    LocalDate.of(2026, 5, 1), LocalDate.of(2026, 5, 3));

// Book
Guest guest = new Guest("Ritesh", "9876543210", "ritesh@email.com");
Booking booking = service.bookRoom(guest, rooms.get(0), 
    LocalDate.of(2026, 5, 1), LocalDate.of(2026, 5, 3));

// Check-in
service.checkIn(booking.getBookingId());

// Add services
booking.addService(new Service("Room Service Dinner", 1500, ServiceType.FOOD));
booking.addService(new Service("Laundry", 500, ServiceType.LAUNDRY));

// Check-out
Bill bill = service.checkOut(booking.getBookingId());
// Total = 2 nights × ₹5000 + ₹1500 + ₹500 = ₹12000
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Cancellation refund policies, Room pricing strategies |
| **Observer** | Notifications on booking/check-in/check-out |
| **State** | Room status transitions: AVAILABLE → BOOKED → OCCUPIED → CLEANING |
| **Factory** | Create rooms based on type |

---

## ⚠️ Edge Cases

- Double booking same room → synchronized booking
- Check-in before booking date → reject
- No-show → mark as NO_SHOW, charge penalty
- Extend stay → check availability for extra days
- Concurrent booking of same room → thread-safe `book()` method

---

## 🔑 Key Takeaways

1. **Date overlap check** is critical for room availability
2. Room status follows a **state machine**: Available → Booked → Occupied → Cleaning → Available
3. **Services** are added during the stay, billed at checkout
4. **Refund policy** is time-based — implements Strategy pattern
5. **Key card management** ties to booking lifecycle
