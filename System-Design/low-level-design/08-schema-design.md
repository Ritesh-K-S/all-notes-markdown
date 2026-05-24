# Schema Design for LLD

Schema design involves defining the **data model** — entities, their attributes, and relationships. This is a critical step in LLD before writing any code.

---

## 1. Entity Identification

### Steps to Identify Entities
1. Read the requirements carefully
2. **Nouns** in the requirements → potential entities
3. **Verbs** → potential methods or relationships
4. **Adjectives** → potential attributes or states

### Example — Hotel Booking System

Requirements:
> "A user can search for hotels by city and date. They can view available rooms and make a booking. Each room has a type (single, double, suite) and a price. Users can cancel bookings. Hotels have a name, address, and star rating."

**Nouns → Entities:** User, Hotel, Room, Booking, City, Address  
**Verbs → Methods:** search, view, book, cancel  
**Adjectives → Attributes:** available, single/double/suite (room type), star rating

---

## 2. Entity-Relationship Design

### Types of Relationships

| Relationship | Example | Mapping |
|-------------|---------|---------|
| **One-to-One** | User ↔ Profile | FK in either table |
| **One-to-Many** | Hotel → Rooms | FK in "many" side |
| **Many-to-Many** | Student ↔ Course | Junction/join table |

### Representing Relationships in Java

#### One-to-One

```java
public class User {
    private String id;
    private String name;
    private String email;
    private UserProfile profile;  // One-to-One
}

public class UserProfile {
    private String id;
    private String bio;
    private String avatarUrl;
    private String phoneNumber;
}
```

#### One-to-Many

```java
public class Hotel {
    private String id;
    private String name;
    private Address address;
    private int starRating;
    private List<Room> rooms;  // One Hotel has many Rooms
}

public class Room {
    private String id;
    private RoomType roomType;
    private double pricePerNight;
    private boolean isAvailable;
    // Reference back to parent (optional)
    private String hotelId;
}
```

#### Many-to-Many

```java
public class Student {
    private String id;
    private String name;
    private List<Enrollment> enrollments;  // Junction entity
}

public class Course {
    private String id;
    private String title;
    private List<Enrollment> enrollments;
}

// Junction entity — can hold extra data
public class Enrollment {
    private String id;
    private Student student;
    private Course course;
    private LocalDate enrollmentDate;
    private String grade;
}
```

---

## 3. Normalization

Normalization eliminates data redundancy and ensures data integrity.

### First Normal Form (1NF)
- Each column has **atomic values** (no lists or nested objects)
- Each row is **unique** (has a primary key)

```
❌ Bad (violates 1NF):
| StudentId | Name  | Courses              |
|-----------|-------|----------------------|
| 1         | Alice | Math, Science, Art   |

✅ Good (1NF):
| StudentId | Name  | Course  |
|-----------|-------|---------|
| 1         | Alice | Math    |
| 1         | Alice | Science |
| 1         | Alice | Art     |
```

### Second Normal Form (2NF)
- Must be in 1NF
- No **partial dependency** — every non-key column depends on the **entire** primary key

```
❌ Bad (partial dependency):
PK = (StudentId, CourseId)
| StudentId | CourseId | StudentName | CourseName |
  StudentName depends only on StudentId (partial dependency)

✅ Good (2NF): Split into separate tables
Students(StudentId, StudentName)
Courses(CourseId, CourseName)
Enrollments(StudentId, CourseId)
```

### Third Normal Form (3NF)
- Must be in 2NF
- No **transitive dependency** — non-key columns don't depend on other non-key columns

```
❌ Bad (transitive dependency):
| EmployeeId | DepartmentId | DepartmentName |
  DepartmentName depends on DepartmentId, not EmployeeId

✅ Good (3NF): Split DepartmentName to Departments table
Employees(EmployeeId, DepartmentId)
Departments(DepartmentId, DepartmentName)
```

---

## 4. Common Schema Patterns

### Enum as Value

```java
public enum RoomType {
    SINGLE, DOUBLE, SUITE, DELUXE
}

public enum BookingStatus {
    PENDING, CONFIRMED, CANCELLED, COMPLETED
}

public enum PaymentStatus {
    INITIATED, SUCCESS, FAILED, REFUNDED
}
```

### Status Tracking with Timestamps

```java
public class Booking {
    private String id;
    private User user;
    private Room room;
    private LocalDate checkIn;
    private LocalDate checkOut;
    private BookingStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private double totalAmount;
}
```

### Audit Fields (Common Base)

```java
public abstract class BaseEntity {
    private String id;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;
}

public class Order extends BaseEntity {
    private String customerId;
    private double amount;
    private OrderStatus status;
}
```

### Soft Delete

```java
public class User extends BaseEntity {
    private String name;
    private String email;
    private boolean isDeleted;       // Soft delete flag
    private LocalDateTime deletedAt;  // When it was deleted
}
```

---

## 5. ID Generation Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **Auto-increment** | 1, 2, 3... | Simple | Not distributed-friendly |
| **UUID** | `550e8400-e29b...` | Globally unique | Large, not sortable |
| **ULID** | `01ARZ3NDEKTSV...` | Sortable + unique | Less common |
| **Snowflake ID** | `123456789012345` | Sortable, distributed | Complex setup |

```java
// UUID example
public class Order {
    private final String id;

    public Order() {
        this.id = UUID.randomUUID().toString();
    }
}
```

---

## 6. Design Patterns in Schema

### Value Object Pattern
Immutable objects identified by their values, not identity.

```java
// Value Object — no ID, compared by value
public class Address {
    private final String street;
    private final String city;
    private final String state;
    private final String zipCode;

    public Address(String street, String city, String state, String zipCode) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Address address = (Address) o;
        return Objects.equals(street, address.street) &&
               Objects.equals(city, address.city) &&
               Objects.equals(state, address.state) &&
               Objects.equals(zipCode, address.zipCode);
    }

    @Override
    public int hashCode() {
        return Objects.hash(street, city, state, zipCode);
    }
}
```

### Money Pattern
Never use `double` for monetary values — use a dedicated Money class.

```java
public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
}
```

---

## 7. Full Schema Design Example — E-Commerce

```java
// === Entities ===

public class User {
    private String id;
    private String name;
    private String email;
    private String passwordHash;
    private List<Address> addresses;
    private LocalDateTime createdAt;
}

public class Product {
    private String id;
    private String name;
    private String description;
    private BigDecimal price;
    private int stockQuantity;
    private Category category;
    private String sellerId;
}

public class Category {
    private String id;
    private String name;
    private Category parentCategory;  // Self-referencing for hierarchy
}

public class Cart {
    private String id;
    private String userId;
    private List<CartItem> items;
}

public class CartItem {
    private String productId;
    private int quantity;
    private BigDecimal priceAtAddition;  // Snapshot price
}

public class Order {
    private String id;
    private String userId;
    private List<OrderItem> items;
    private Address shippingAddress;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private Payment payment;
    private LocalDateTime orderDate;
}

public class OrderItem {
    private String productId;
    private String productName;  // Snapshot — product name at time of order
    private int quantity;
    private BigDecimal unitPrice;  // Snapshot — price at time of order
}

public class Payment {
    private String id;
    private String orderId;
    private BigDecimal amount;
    private PaymentMethod method;
    private PaymentStatus status;
    private String transactionId;
    private LocalDateTime paidAt;
}
```

### Relationship Summary

```
User  ──1:N──>  Order
User  ──1:1──>  Cart
Cart  ──1:N──>  CartItem
Order ──1:N──>  OrderItem
Order ──1:1──>  Payment
Product ──N:1──> Category
Category ──N:1──> Category (self-ref)
```

---

## 8. Schema Design Checklist

- [ ] All entities identified from requirements
- [ ] Primary keys defined for each entity
- [ ] Relationships mapped (1:1, 1:N, N:M)
- [ ] Enums used for fixed values (status, type)
- [ ] Timestamps added (createdAt, updatedAt)
- [ ] Data snapshots where needed (e.g., price at order time)
- [ ] Normalized to at least 3NF
- [ ] Value objects identified (Address, Money)
- [ ] Audit fields added where appropriate
- [ ] Soft delete considered for critical entities
