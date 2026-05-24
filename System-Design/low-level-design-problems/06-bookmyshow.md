# 06 - BookMyShow / Movie Ticket Booking System

## 📋 Problem Statement

Design an online movie ticket booking system that can:
- Allow users to browse movies, theaters, and shows
- Book seats with concurrent access handling
- Process payments and generate tickets

---

## 📌 Requirements

### Functional Requirements
1. **List movies** playing in a city
2. **List theaters** showing a specific movie
3. **Browse available seats** for a show
4. **Book tickets** — select seats, make payment
5. **Cancel booking** and process refund
6. Handle **concurrent seat booking** (no double booking)
7. **Temporary hold** on seats during payment (5-minute timeout)
8. Support different **seat types**: Regular, Premium, VIP

### Non-Functional Requirements
- Handle **high concurrency** during popular movie releases
- Seats must be **locked** during booking process
- Payment processing with retry mechanism
- Notification on booking confirmation

---

## 🧩 Core Entities

```
Movie, Theater, Screen, Show, Seat, Booking, Payment, 
User, City, SeatType, ShowSeat, Notification
```

---

## 📐 Class Diagram

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│    City       │────▶│     Theater       │────▶│    Screen     │
│  - name       │1  * │  - name           │1  * │  - screenId   │
│  - state      │     │  - address        │     │  - seats      │
│  - theaters[] │     │  - screens[]      │     │  - totalSeats │
└──────────────┘     └──────────────────┘     └──────┬───────┘
                                                      │ 1..*
┌──────────────┐     ┌──────────────────┐     ┌──────▼───────┐
│    Movie      │────▶│      Show         │────▶│     Seat      │
│  - title      │1  * │  - showId         │1  * │  - seatId     │
│  - genre      │     │  - startTime      │     │  - row        │
│  - duration   │     │  - endTime        │     │  - col        │
│  - rating     │     │  - screen         │     │  - type       │
│  - language   │     │  - movie          │     └──────────────┘
└──────────────┘     └────────┬─────────┘
                              │ 1..*
                     ┌────────▼─────────┐
                     │    ShowSeat       │    ← per show seat status
                     │  - show           │
                     │  - seat           │
                     │  - status         │
                     │  - price          │
                     │  - lockedAt       │
                     │  - lockedBy       │
                     └──────────────────┘

┌──────────────────────────────────────────────────────┐
│                     Booking                           │
│  - bookingId: String                                  │
│  - user: User                                         │
│  - show: Show                                         │
│  - seats: List<ShowSeat>                              │
│  - totalAmount: double                                │
│  - status: BookingStatus                              │
│  - payment: Payment                                   │
│  - createdAt: LocalDateTime                           │
│  + confirm(): void                                    │
│  + cancel(): void                                     │
└──────────────────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum SeatType {
    REGULAR, PREMIUM, VIP
}

public enum SeatStatus {
    AVAILABLE, LOCKED, BOOKED
}

public enum BookingStatus {
    PENDING, CONFIRMED, CANCELLED, EXPIRED
}

public enum PaymentStatus {
    PENDING, SUCCESS, FAILED, REFUNDED
}
```

---

## 💻 Code Implementation

### Core Models

```java
public class Movie {
    private String movieId;
    private String title;
    private String genre;
    private int durationMinutes;
    private String language;
    private double rating;
    private LocalDate releaseDate;
}

public class Theater {
    private String theaterId;
    private String name;
    private String address;
    private City city;
    private List<Screen> screens;
}

public class Screen {
    private String screenId;
    private String name;
    private List<Seat> seats;
    private Theater theater;
}

public class Show {
    private String showId;
    private Movie movie;
    private Screen screen;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private Map<String, ShowSeat> showSeats; // seatId → ShowSeat
}

public class Seat {
    private String seatId;
    private int row;
    private int col;
    private SeatType type;
}
```

### ShowSeat (Per-Show Seat Status) — Critical Class

```java
public class ShowSeat {
    private Show show;
    private Seat seat;
    private SeatStatus status;
    private double price;
    private LocalDateTime lockedAt;
    private String lockedByUserId;

    private static final long LOCK_TIMEOUT_MINUTES = 5;

    public ShowSeat(Show show, Seat seat, double price) {
        this.show = show;
        this.seat = seat;
        this.price = price;
        this.status = SeatStatus.AVAILABLE;
    }

    public synchronized boolean lock(String userId) {
        // Release expired locks
        if (status == SeatStatus.LOCKED && isLockExpired()) {
            release();
        }

        if (status != SeatStatus.AVAILABLE) return false;

        this.status = SeatStatus.LOCKED;
        this.lockedAt = LocalDateTime.now();
        this.lockedByUserId = userId;
        return true;
    }

    public synchronized void book() {
        if (status != SeatStatus.LOCKED) {
            throw new IllegalStateException("Seat must be locked before booking");
        }
        this.status = SeatStatus.BOOKED;
    }

    public synchronized void release() {
        this.status = SeatStatus.AVAILABLE;
        this.lockedAt = null;
        this.lockedByUserId = null;
    }

    private boolean isLockExpired() {
        return lockedAt != null &&
               Duration.between(lockedAt, LocalDateTime.now()).toMinutes() >= LOCK_TIMEOUT_MINUTES;
    }

    public double getPrice() { return price; }
    public SeatStatus getStatus() { return status; }
}
```

### Booking Service

```java
public class BookingService {
    private Map<String, Booking> bookings;
    private PaymentService paymentService;
    private NotificationService notificationService;

    public BookingService() {
        this.bookings = new ConcurrentHashMap<>();
        this.paymentService = new PaymentService();
        this.notificationService = new NotificationService();
    }

    public Booking createBooking(User user, Show show, List<String> seatIds) {
        // Step 1: Lock all requested seats
        List<ShowSeat> lockedSeats = new ArrayList<>();
        try {
            for (String seatId : seatIds) {
                ShowSeat showSeat = show.getShowSeat(seatId);
                if (showSeat == null || !showSeat.lock(user.getUserId())) {
                    // Rollback previously locked seats
                    lockedSeats.forEach(ShowSeat::release);
                    throw new SeatNotAvailableException("Seat " + seatId + " is not available");
                }
                lockedSeats.add(showSeat);
            }

            // Step 2: Calculate total amount
            double totalAmount = lockedSeats.stream()
                                            .mapToDouble(ShowSeat::getPrice)
                                            .sum();

            // Step 3: Create booking
            Booking booking = new Booking(user, show, lockedSeats, totalAmount);
            bookings.put(booking.getBookingId(), booking);

            return booking;

        } catch (Exception e) {
            lockedSeats.forEach(ShowSeat::release);
            throw e;
        }
    }

    public void confirmBooking(String bookingId, PaymentMethod paymentMethod) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) throw new BookingNotFoundException("Booking not found");

        // Process payment
        Payment payment = paymentService.processPayment(booking.getTotalAmount(), paymentMethod);

        if (payment.getStatus() == PaymentStatus.SUCCESS) {
            booking.confirm(payment);
            booking.getSeats().forEach(ShowSeat::book);

            // Send confirmation
            notificationService.sendBookingConfirmation(booking);
        } else {
            booking.getSeats().forEach(ShowSeat::release);
            booking.setStatus(BookingStatus.EXPIRED);
            throw new PaymentFailedException("Payment failed");
        }
    }

    public void cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null || booking.getStatus() != BookingStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot cancel this booking");
        }

        // Release seats
        booking.getSeats().forEach(ShowSeat::release);
        booking.setStatus(BookingStatus.CANCELLED);

        // Process refund
        paymentService.processRefund(booking.getPayment());
        notificationService.sendCancellationNotification(booking);
    }
}
```

### Booking Class

```java
public class Booking {
    private String bookingId;
    private User user;
    private Show show;
    private List<ShowSeat> seats;
    private double totalAmount;
    private BookingStatus status;
    private Payment payment;
    private LocalDateTime createdAt;

    public Booking(User user, Show show, List<ShowSeat> seats, double totalAmount) {
        this.bookingId = UUID.randomUUID().toString();
        this.user = user;
        this.show = show;
        this.seats = seats;
        this.totalAmount = totalAmount;
        this.status = BookingStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }

    public void confirm(Payment payment) {
        this.payment = payment;
        this.status = BookingStatus.CONFIRMED;
    }

    // Getters
    public String getBookingId() { return bookingId; }
    public BookingStatus getStatus() { return status; }
    public double getTotalAmount() { return totalAmount; }
    public List<ShowSeat> getSeats() { return seats; }
}
```

### Search / Browse Service

```java
public class MovieSearchService {
    private Map<City, List<Movie>> cityMovies;
    private Map<Movie, List<Show>> movieShows;

    public List<Movie> getMoviesByCity(City city) {
        return cityMovies.getOrDefault(city, Collections.emptyList());
    }

    public List<Show> getShowsByMovie(Movie movie, City city) {
        return movieShows.getOrDefault(movie, Collections.emptyList())
                         .stream()
                         .filter(show -> show.getScreen().getTheater().getCity().equals(city))
                         .collect(Collectors.toList());
    }

    public List<ShowSeat> getAvailableSeats(Show show) {
        return show.getShowSeats().values().stream()
                   .filter(s -> s.getStatus() == SeatStatus.AVAILABLE)
                   .collect(Collectors.toList());
    }
}
```

---

## 🧪 Usage Flow

```java
// 1. User searches movies in a city
List<Movie> movies = searchService.getMoviesByCity(bangalore);

// 2. User selects a movie and sees shows
List<Show> shows = searchService.getShowsByMovie(avengers, bangalore);

// 3. User selects a show and browses seats
List<ShowSeat> seats = searchService.getAvailableSeats(show);

// 4. User selects seats and creates booking
Booking booking = bookingService.createBooking(user, show, List.of("A1", "A2"));

// 5. User pays
bookingService.confirmBooking(booking.getBookingId(), PaymentMethod.UPI);

// 6. User cancels (optional)
bookingService.cancelBooking(booking.getBookingId());
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Payment methods (UPI, Card, Wallet) |
| **Observer** | Notifications on booking/cancellation |
| **Singleton** | BookingService, SearchService |
| **Factory** | Creating Seat objects based on type |

---

## 🔒 Concurrency Handling

```
1. ShowSeat.lock() is synchronized
   → Only one thread can lock a seat at a time

2. Lock timeout (5 min)
   → If payment not completed, lock expires, seat becomes available

3. Rollback on partial failure
   → If locking seat 3 fails, seats 1 & 2 are released

4. ConcurrentHashMap for bookings
   → Thread-safe booking storage
```

---

## ⚠️ Edge Cases

- Two users try to book same seat simultaneously → lock mechanism
- Payment fails after seats locked → release seats
- User doesn't complete payment within 5 min → auto-release
- Cancel after show starts → reject cancellation
- System crash during booking → seats remain locked until timeout

---

## 🔑 Key Takeaways

1. **ShowSeat** is the critical entity — separates physical seat from per-show status
2. **Lock + Timeout** pattern handles concurrent booking
3. **Rollback** on any failure in the booking flow
4. Separate **Booking creation** from **Payment confirmation** (two-step)
5. Pricing varies by seat type and show time
