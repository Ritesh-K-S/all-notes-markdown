# 13 - Online Shopping System (Amazon)

## 📋 Problem Statement

Design an online shopping platform that supports:
- Product catalog with search and filtering
- Shopping cart management
- Order placement with payment processing
- Order tracking and delivery

---

## 📌 Requirements

### Functional Requirements
1. **Browse/Search** products by name, category, price range
2. **Product details** — name, price, description, reviews, stock
3. **Shopping cart** — add, remove, update quantity
4. **Checkout** — create order from cart
5. **Payment** — multiple methods (Card, UPI, Wallet, COD)
6. **Order tracking** — status updates
7. **Reviews and ratings** on products
8. **Inventory management** — stock tracking
9. **Notifications** — order confirmation, shipping updates

### Non-Functional Requirements
- Handle concurrent purchase of last item
- Scalable product catalog
- Order consistency (no overselling)

---

## 🧩 Core Entities

```
User, Product, Category, Cart, CartItem, Order, OrderItem,
Payment, Address, Review, Inventory, ShippingInfo
```

---

## 📐 Class Diagram

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│    User       │     │    Product        │     │   Category    │
│  - userId     │     │  - productId      │     │  - name       │
│  - name       │     │  - name           │     │  - parent     │
│  - email      │     │  - price          │     │  - children[] │
│  - addresses[]│     │  - description    │     └──────────────┘
│  - cart: Cart │     │  - category       │
│  - orders[]   │     │  - stock          │
└──────┬───────┘     │  - reviews[]      │
       │              └──────────────────┘
       │
┌──────▼───────────────────────────────────────────────┐
│                      Cart                             │
│  - cartId: String                                     │
│  - items: Map<String, CartItem>                       │
│  + addItem(product, qty): void                        │
│  + removeItem(productId): void                        │
│  + updateQuantity(productId, qty): void               │
│  + getTotal(): double                                 │
│  + clear(): void                                      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                      Order                            │
│  - orderId: String                                    │
│  - user: User                                         │
│  - items: List<OrderItem>                             │
│  - totalAmount: double                                │
│  - status: OrderStatus                                │
│  - shippingAddress: Address                           │
│  - payment: Payment                                   │
│  - createdAt: LocalDateTime                           │
│  + updateStatus(status): void                         │
└──────────────────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum OrderStatus {
    PLACED, CONFIRMED, SHIPPED, OUT_FOR_DELIVERY, DELIVERED, CANCELLED, RETURNED
}

public enum PaymentMethod {
    CREDIT_CARD, DEBIT_CARD, UPI, WALLET, COD
}

public enum PaymentStatus {
    PENDING, SUCCESS, FAILED, REFUNDED
}
```

---

## 💻 Code Implementation

### Product and Inventory

```java
public class Product {
    private String productId;
    private String name;
    private String description;
    private double price;
    private Category category;
    private double rating;
    private List<Review> reviews;
    private Inventory inventory;

    public Product(String productId, String name, double price, Category category, int stock) {
        this.productId = productId;
        this.name = name;
        this.price = price;
        this.category = category;
        this.reviews = new ArrayList<>();
        this.inventory = new Inventory(stock);
    }

    public void addReview(Review review) {
        reviews.add(review);
        this.rating = reviews.stream().mapToDouble(Review::getRating).average().orElse(0);
    }

    public boolean isInStock() { return inventory.getStock() > 0; }
    public String getProductId() { return productId; }
    public double getPrice() { return price; }
    public Inventory getInventory() { return inventory; }
}

public class Inventory {
    private int stock;

    public Inventory(int stock) {
        this.stock = stock;
    }

    public synchronized boolean reserve(int quantity) {
        if (stock >= quantity) {
            stock -= quantity;
            return true;
        }
        return false;
    }

    public synchronized void release(int quantity) {
        stock += quantity;
    }

    public int getStock() { return stock; }
}
```

### Cart and CartItem

```java
public class CartItem {
    private Product product;
    private int quantity;

    public CartItem(Product product, int quantity) {
        this.product = product;
        this.quantity = quantity;
    }

    public double getSubtotal() {
        return product.getPrice() * quantity;
    }

    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
}

public class Cart {
    private String cartId;
    private Map<String, CartItem> items; // productId → CartItem

    public Cart() {
        this.cartId = UUID.randomUUID().toString();
        this.items = new LinkedHashMap<>();
    }

    public void addItem(Product product, int quantity) {
        String productId = product.getProductId();
        if (items.containsKey(productId)) {
            CartItem item = items.get(productId);
            item.setQuantity(item.getQuantity() + quantity);
        } else {
            items.put(productId, new CartItem(product, quantity));
        }
    }

    public void removeItem(String productId) {
        items.remove(productId);
    }

    public void updateQuantity(String productId, int quantity) {
        CartItem item = items.get(productId);
        if (item != null) {
            if (quantity <= 0) {
                items.remove(productId);
            } else {
                item.setQuantity(quantity);
            }
        }
    }

    public double getTotal() {
        return items.values().stream().mapToDouble(CartItem::getSubtotal).sum();
    }

    public void clear() {
        items.clear();
    }

    public Map<String, CartItem> getItems() { return items; }
    public boolean isEmpty() { return items.isEmpty(); }
}
```

### Order and OrderItem

```java
public class OrderItem {
    private Product product;
    private int quantity;
    private double priceAtOrder; // snapshot of price at order time

    public OrderItem(Product product, int quantity) {
        this.product = product;
        this.quantity = quantity;
        this.priceAtOrder = product.getPrice();
    }

    public double getSubtotal() { return priceAtOrder * quantity; }
}

public class Order {
    private String orderId;
    private User user;
    private List<OrderItem> items;
    private double totalAmount;
    private OrderStatus status;
    private Address shippingAddress;
    private Payment payment;
    private LocalDateTime createdAt;

    public Order(User user, List<OrderItem> items, Address shippingAddress) {
        this.orderId = UUID.randomUUID().toString();
        this.user = user;
        this.items = items;
        this.totalAmount = items.stream().mapToDouble(OrderItem::getSubtotal).sum();
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.PLACED;
        this.createdAt = LocalDateTime.now();
    }

    public void updateStatus(OrderStatus status) {
        this.status = status;
    }

    // Getters
    public String getOrderId() { return orderId; }
    public OrderStatus getStatus() { return status; }
    public double getTotalAmount() { return totalAmount; }
    public List<OrderItem> getItems() { return items; }
}
```

### Order Service (Controller)

```java
public class OrderService {
    private Map<String, Order> orders;
    private PaymentService paymentService;
    private NotificationService notificationService;

    public OrderService() {
        this.orders = new ConcurrentHashMap<>();
        this.paymentService = new PaymentService();
        this.notificationService = new NotificationService();
    }

    public Order placeOrder(User user, Address shippingAddress, PaymentMethod paymentMethod) {
        Cart cart = user.getCart();
        if (cart.isEmpty()) throw new EmptyCartException("Cart is empty");

        // Step 1: Reserve inventory for all items
        List<CartItem> reservedItems = new ArrayList<>();
        try {
            for (CartItem item : cart.getItems().values()) {
                if (!item.getProduct().getInventory().reserve(item.getQuantity())) {
                    // Rollback previously reserved
                    rollbackReservations(reservedItems);
                    throw new OutOfStockException(item.getProduct().getName() + " is out of stock");
                }
                reservedItems.add(item);
            }
        } catch (OutOfStockException e) {
            throw e;
        }

        // Step 2: Create order items (snapshot prices)
        List<OrderItem> orderItems = cart.getItems().values().stream()
            .map(ci -> new OrderItem(ci.getProduct(), ci.getQuantity()))
            .collect(Collectors.toList());

        // Step 3: Create order
        Order order = new Order(user, orderItems, shippingAddress);

        // Step 4: Process payment
        Payment payment = paymentService.processPayment(order.getTotalAmount(), paymentMethod);
        if (payment.getStatus() != PaymentStatus.SUCCESS) {
            rollbackReservations(reservedItems);
            throw new PaymentFailedException("Payment failed");
        }

        order.setPayment(payment);
        order.updateStatus(OrderStatus.CONFIRMED);
        orders.put(order.getOrderId(), order);

        // Step 5: Clear cart and notify
        cart.clear();
        user.addOrder(order);
        notificationService.sendOrderConfirmation(user, order);

        return order;
    }

    public void cancelOrder(String orderId) {
        Order order = orders.get(orderId);
        if (order == null) throw new OrderNotFoundException("Order not found");
        if (order.getStatus() == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel delivered order");
        }

        // Release inventory
        for (OrderItem item : order.getItems()) {
            item.getProduct().getInventory().release(item.getQuantity());
        }

        order.updateStatus(OrderStatus.CANCELLED);
        paymentService.processRefund(order.getPayment());
        notificationService.sendCancellationNotification(order.getUser(), order);
    }

    public void updateOrderStatus(String orderId, OrderStatus status) {
        Order order = orders.get(orderId);
        if (order != null) {
            order.updateStatus(status);
            notificationService.sendStatusUpdate(order.getUser(), order);
        }
    }

    private void rollbackReservations(List<CartItem> items) {
        for (CartItem item : items) {
            item.getProduct().getInventory().release(item.getQuantity());
        }
    }
}
```

### Search Service

```java
public class ProductSearchService {
    private Map<String, Product> productIndex;
    private Map<String, List<Product>> categoryIndex;

    public List<Product> searchByName(String keyword) {
        return productIndex.values().stream()
            .filter(p -> p.getName().toLowerCase().contains(keyword.toLowerCase()))
            .collect(Collectors.toList());
    }

    public List<Product> searchByCategory(String category) {
        return categoryIndex.getOrDefault(category, Collections.emptyList());
    }

    public List<Product> filterByPriceRange(List<Product> products, double min, double max) {
        return products.stream()
            .filter(p -> p.getPrice() >= min && p.getPrice() <= max)
            .collect(Collectors.toList());
    }

    public List<Product> sortByPrice(List<Product> products, boolean ascending) {
        return products.stream()
            .sorted(ascending ? Comparator.comparingDouble(Product::getPrice)
                              : Comparator.comparingDouble(Product::getPrice).reversed())
            .collect(Collectors.toList());
    }

    public List<Product> sortByRating(List<Product> products) {
        return products.stream()
            .sorted(Comparator.comparingDouble(Product::getRating).reversed())
            .collect(Collectors.toList());
    }
}
```

---

## 🧪 Usage Example

```java
// Setup
ProductSearchService searchService = new ProductSearchService();
OrderService orderService = new OrderService();

Product phone = new Product("P1", "iPhone 15", 79999, electronics, 50);
Product laptop = new Product("P2", "MacBook Pro", 149999, electronics, 20);

User user = new User("U1", "Ritesh", "ritesh@email.com");
Address address = new Address("123 MG Road", "Bangalore", "560001");

// Browse and Search
List<Product> results = searchService.searchByName("iPhone");

// Add to cart
user.getCart().addItem(phone, 1);
user.getCart().addItem(laptop, 1);
System.out.println("Cart Total: ₹" + user.getCart().getTotal()); // ₹229998

// Place order
Order order = orderService.placeOrder(user, address, PaymentMethod.UPI);
// → Order confirmed, cart cleared, notification sent

// Track order
orderService.updateOrderStatus(order.getOrderId(), OrderStatus.SHIPPED);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Payment methods, Sorting strategies |
| **Observer** | Order status notifications |
| **Factory** | Creating payment processors |
| **Repository** | ProductSearchService with indexes |

---

## ⚠️ Edge Cases

- Two users buy last item simultaneously → synchronized `reserve()`
- Payment fails after inventory reserved → rollback reservations
- Price changes after adding to cart → `priceAtOrder` snapshot
- Cart item out of stock during checkout → inform user
- Cancel after shipping → reject or initiate return

---

## 🔑 Key Takeaways

1. **Inventory.reserve()** is synchronized — prevents overselling
2. **Price snapshot** in OrderItem — price at order time, not current price
3. **Rollback pattern** — if any step fails, undo previous steps
4. **Cart → Order** is a two-step process: create order, then pay
5. Order status follows a **linear state machine**
