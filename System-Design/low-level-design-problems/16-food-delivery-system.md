# 16 - Food Delivery System (Zomato/Swiggy)

## 📋 Problem Statement

Design a food delivery system that:
- Allows users to browse restaurants and menus
- Handles order placement and delivery assignment
- Tracks order status from placed to delivered

---

## 📌 Requirements

### Functional Requirements
1. **Browse restaurants** by location, cuisine, rating
2. **View menu** with items and prices
3. **Place order** — select items, apply coupons
4. **Assign delivery agent** based on proximity
5. **Track order** status in real-time
6. **Payment** — prepaid or COD
7. **Ratings** for restaurant and delivery agent
8. **Order history**

### Non-Functional Requirements
- Handle concurrent orders
- Efficient delivery agent assignment
- Real-time status updates

---

## 🧩 Core Entities

```
Customer, Restaurant, MenuItem, Order, OrderItem, 
DeliveryAgent, Payment, Coupon, Rating, Location
```

---

## 🔧 Enums

```java
public enum OrderStatus {
    PLACED, CONFIRMED, PREPARING, READY_FOR_PICKUP,
    PICKED_UP, ON_THE_WAY, DELIVERED, CANCELLED
}

public enum CuisineType {
    NORTH_INDIAN, SOUTH_INDIAN, CHINESE, ITALIAN, FAST_FOOD, DESSERTS
}

public enum AgentStatus {
    AVAILABLE, ON_DELIVERY, OFFLINE
}
```

---

## 💻 Code Implementation

### Restaurant and Menu

```java
public class Restaurant {
    private String restaurantId;
    private String name;
    private Location location;
    private List<MenuItem> menu;
    private CuisineType cuisine;
    private double rating;
    private boolean isOpen;

    public Restaurant(String id, String name, Location location, CuisineType cuisine) {
        this.restaurantId = id;
        this.name = name;
        this.location = location;
        this.cuisine = cuisine;
        this.menu = new ArrayList<>();
        this.rating = 4.0;
        this.isOpen = true;
    }

    public void addMenuItem(MenuItem item) { menu.add(item); }
    public List<MenuItem> getMenu() { return menu; }
    public Location getLocation() { return location; }
    public boolean isOpen() { return isOpen; }
}

public class MenuItem {
    private String itemId;
    private String name;
    private double price;
    private String description;
    private boolean isVeg;
    private boolean isAvailable;

    public MenuItem(String itemId, String name, double price, boolean isVeg) {
        this.itemId = itemId;
        this.name = name;
        this.price = price;
        this.isVeg = isVeg;
        this.isAvailable = true;
    }

    public double getPrice() { return price; }
    public boolean isAvailable() { return isAvailable; }
}
```

### Order Classes

```java
public class OrderItem {
    private MenuItem menuItem;
    private int quantity;

    public OrderItem(MenuItem menuItem, int quantity) {
        this.menuItem = menuItem;
        this.quantity = quantity;
    }

    public double getSubtotal() { return menuItem.getPrice() * quantity; }
}

public class Order {
    private String orderId;
    private Customer customer;
    private Restaurant restaurant;
    private List<OrderItem> items;
    private DeliveryAgent agent;
    private OrderStatus status;
    private double totalAmount;
    private double deliveryFee;
    private double discount;
    private Location deliveryAddress;
    private LocalDateTime orderTime;
    private LocalDateTime deliveryTime;

    public Order(Customer customer, Restaurant restaurant, 
                 List<OrderItem> items, Location deliveryAddress) {
        this.orderId = UUID.randomUUID().toString();
        this.customer = customer;
        this.restaurant = restaurant;
        this.items = items;
        this.deliveryAddress = deliveryAddress;
        this.status = OrderStatus.PLACED;
        this.orderTime = LocalDateTime.now();
        this.deliveryFee = calculateDeliveryFee();
        this.totalAmount = calculateTotal();
    }

    private double calculateDeliveryFee() {
        double distance = restaurant.getLocation().distanceTo(deliveryAddress);
        if (distance <= 3) return 20;
        if (distance <= 7) return 40;
        return 60;
    }

    private double calculateTotal() {
        double itemTotal = items.stream().mapToDouble(OrderItem::getSubtotal).sum();
        return itemTotal + deliveryFee - discount;
    }

    public void applyCoupon(Coupon coupon) {
        this.discount = coupon.calculateDiscount(totalAmount + discount);
        this.totalAmount = calculateTotal();
    }

    public void assignAgent(DeliveryAgent agent) {
        this.agent = agent;
        this.status = OrderStatus.CONFIRMED;
    }

    public void updateStatus(OrderStatus status) {
        this.status = status;
        if (status == OrderStatus.DELIVERED) {
            this.deliveryTime = LocalDateTime.now();
        }
    }

    // Getters
    public String getOrderId() { return orderId; }
    public OrderStatus getStatus() { return status; }
    public double getTotalAmount() { return totalAmount; }
    public Restaurant getRestaurant() { return restaurant; }
    public Location getDeliveryAddress() { return deliveryAddress; }
}
```

### Delivery Agent Assignment

```java
public class DeliveryAgent {
    private String agentId;
    private String name;
    private Location location;
    private AgentStatus status;
    private double rating;

    public boolean assignOrder() {
        if (status != AgentStatus.AVAILABLE) return false;
        this.status = AgentStatus.ON_DELIVERY;
        return true;
    }

    public void completeDelivery() {
        this.status = AgentStatus.AVAILABLE;
    }

    public Location getLocation() { return location; }
    public AgentStatus getStatus() { return status; }
}

public class DeliveryAssignmentService {
    private List<DeliveryAgent> agents;

    public DeliveryAgent assignAgent(Location restaurantLocation) {
        return agents.stream()
            .filter(a -> a.getStatus() == AgentStatus.AVAILABLE)
            .min(Comparator.comparingDouble(a -> a.getLocation().distanceTo(restaurantLocation)))
            .filter(a -> a.assignOrder())
            .orElseThrow(() -> new NoAgentAvailableException("No delivery agents available"));
    }
}
```

### Order Service

```java
public class OrderService {
    private Map<String, Order> orders;
    private DeliveryAssignmentService deliveryService;
    private NotificationService notificationService;

    public Order placeOrder(Customer customer, Restaurant restaurant, 
                            List<OrderItem> items, Location deliveryAddress) {
        if (!restaurant.isOpen()) throw new RestaurantClosedException("Restaurant is closed");

        // Validate items
        for (OrderItem item : items) {
            if (!item.getMenuItem().isAvailable()) {
                throw new ItemNotAvailableException(item.getMenuItem().getName() + " is not available");
            }
        }

        Order order = new Order(customer, restaurant, items, deliveryAddress);

        // Assign delivery agent
        DeliveryAgent agent = deliveryService.assignAgent(restaurant.getLocation());
        order.assignAgent(agent);

        orders.put(order.getOrderId(), order);
        notificationService.notifyCustomer(customer, "Order placed! ID: " + order.getOrderId());
        notificationService.notifyRestaurant(restaurant, "New order received: " + order.getOrderId());

        return order;
    }

    public void updateOrderStatus(String orderId, OrderStatus status) {
        Order order = orders.get(orderId);
        order.updateStatus(status);

        if (status == OrderStatus.DELIVERED) {
            order.getAgent().completeDelivery();
        }
    }
}
```

### Coupon System (Strategy Pattern)

```java
public interface Coupon {
    double calculateDiscount(double orderTotal);
    boolean isValid(double orderTotal);
}

public class PercentageCoupon implements Coupon {
    private String code;
    private double percentage;
    private double maxDiscount;
    private double minOrderAmount;

    @Override
    public double calculateDiscount(double orderTotal) {
        if (!isValid(orderTotal)) return 0;
        return Math.min(orderTotal * percentage / 100, maxDiscount);
    }

    @Override
    public boolean isValid(double orderTotal) {
        return orderTotal >= minOrderAmount;
    }
}

public class FlatCoupon implements Coupon {
    private String code;
    private double discountAmount;
    private double minOrderAmount;

    @Override
    public double calculateDiscount(double orderTotal) {
        if (!isValid(orderTotal)) return 0;
        return discountAmount;
    }

    @Override
    public boolean isValid(double orderTotal) {
        return orderTotal >= minOrderAmount;
    }
}
```

---

## 🧪 Usage Example

```java
Restaurant restaurant = new Restaurant("R1", "Pizza Palace", new Location(12.97, 77.59), CuisineType.ITALIAN);
restaurant.addMenuItem(new MenuItem("M1", "Margherita Pizza", 299, true));
restaurant.addMenuItem(new MenuItem("M2", "Pepperoni Pizza", 399, false));

Customer customer = new Customer("C1", "Ritesh");
Location deliveryAddr = new Location(12.96, 77.58);

List<OrderItem> items = List.of(
    new OrderItem(restaurant.getMenu().get(0), 2),  // 2 Margherita
    new OrderItem(restaurant.getMenu().get(1), 1)   // 1 Pepperoni
);

OrderService service = new OrderService();
Order order = service.placeOrder(customer, restaurant, items, deliveryAddr);
// Total: (299×2 + 399) + delivery fee - discount

// Status updates
service.updateOrderStatus(order.getOrderId(), OrderStatus.PREPARING);
service.updateOrderStatus(order.getOrderId(), OrderStatus.PICKED_UP);
service.updateOrderStatus(order.getOrderId(), OrderStatus.DELIVERED);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Coupon types, Delivery assignment, Pricing |
| **Observer** | Order status notifications |
| **State** | Order status transitions |
| **Factory** | Creating different coupon types |

---

## 🔑 Key Takeaways

1. **Order status** follows a defined state machine
2. **Delivery assignment** uses nearest-agent strategy
3. **Delivery fee** is distance-based
4. **Coupon system** uses Strategy pattern (percentage vs flat)
5. Separate concerns: OrderService, DeliveryService, NotificationService
