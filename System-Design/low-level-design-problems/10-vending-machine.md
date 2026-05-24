# 10 - Vending Machine

## 📋 Problem Statement

Design a vending machine that:
- Displays products with prices
- Accepts coins/notes, tracks inserted amount
- Dispenses product and returns change
- Uses State Pattern for different machine states

---

## 📌 Requirements

### Functional Requirements
1. Display **available products** with prices
2. User inserts **coins/notes** (1, 2, 5, 10, 20, 50, 100)
3. User **selects a product**
4. Machine **validates** sufficient payment
5. Machine **dispenses product** and **returns change**
6. Support **cancel** — return all inserted money
7. Admin can **restock** products and **collect cash**

### Non-Functional Requirements
- One transaction at a time
- Robust state management
- Handle edge cases (out of stock, insufficient change)

---

## 🧩 Core Entities

```
VendingMachine, Product, Slot, Coin, State (interface), 
IdleState, HasMoneyState, DispensingState, Inventory
```

---

## 📐 State Diagram

```
         insertMoney()           selectProduct()         dispense()
  IDLE ──────────────▶ HAS_MONEY ──────────────▶ DISPENSING ──────▶ IDLE
   │                      │                          │
   │                      │ cancel()                  │ error
   │                      ▼                           ▼
   │                    IDLE                     HAS_MONEY
   │
   │  selectProduct() → Error: Insert money first
```

---

## 💻 Code Implementation

### Product and Slot

```java
public class Product {
    private String name;
    private double price;

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    public String getName() { return name; }
    public double getPrice() { return price; }
}

public class Slot {
    private int slotNumber;
    private Product product;
    private int quantity;

    public Slot(int slotNumber, Product product, int quantity) {
        this.slotNumber = slotNumber;
        this.product = product;
        this.quantity = quantity;
    }

    public boolean isAvailable() {
        return quantity > 0;
    }

    public void dispense() {
        if (quantity <= 0) throw new OutOfStockException("Slot " + slotNumber + " is empty");
        quantity--;
    }

    public void restock(int count) {
        this.quantity += count;
    }

    // Getters
    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
    public int getSlotNumber() { return slotNumber; }
}
```

### Coin Enum

```java
public enum Coin {
    ONE(1), TWO(2), FIVE(5), TEN(10), TWENTY(20), FIFTY(50), HUNDRED(100);

    private final int value;

    Coin(int value) { this.value = value; }
    public int getValue() { return value; }
}
```

### State Interface

```java
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, Coin coin);
    void selectProduct(VendingMachine machine, int slotNumber);
    void dispenseProduct(VendingMachine machine);
    void cancel(VendingMachine machine);
}
```

### Idle State

```java
public class IdleState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, Coin coin) {
        machine.addBalance(coin.getValue());
        System.out.println("Inserted: ₹" + coin.getValue() + " | Balance: ₹" + machine.getBalance());
        machine.setState(new HasMoneyState());
    }

    @Override
    public void selectProduct(VendingMachine machine, int slotNumber) {
        System.out.println("Please insert money first!");
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        System.out.println("Please insert money and select product first!");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("No transaction to cancel.");
    }
}
```

### HasMoney State

```java
public class HasMoneyState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, Coin coin) {
        machine.addBalance(coin.getValue());
        System.out.println("Inserted: ₹" + coin.getValue() + " | Balance: ₹" + machine.getBalance());
    }

    @Override
    public void selectProduct(VendingMachine machine, int slotNumber) {
        Slot slot = machine.getSlot(slotNumber);

        if (slot == null) {
            System.out.println("Invalid slot number!");
            return;
        }

        if (!slot.isAvailable()) {
            System.out.println("Product out of stock! Select another or cancel.");
            return;
        }

        if (machine.getBalance() < slot.getProduct().getPrice()) {
            System.out.println("Insufficient balance! Need ₹" + slot.getProduct().getPrice()
                + ", have ₹" + machine.getBalance() + ". Insert more or cancel.");
            return;
        }

        machine.setSelectedSlot(slotNumber);
        machine.setState(new DispensingState());
        machine.dispenseProduct();
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        System.out.println("Please select a product first!");
    }

    @Override
    public void cancel(VendingMachine machine) {
        double refund = machine.getBalance();
        machine.resetBalance();
        machine.setState(new IdleState());
        System.out.println("Transaction cancelled. Refund: ₹" + refund);
    }
}
```

### Dispensing State

```java
public class DispensingState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, Coin coin) {
        System.out.println("Please wait, dispensing in progress...");
    }

    @Override
    public void selectProduct(VendingMachine machine, int slotNumber) {
        System.out.println("Please wait, dispensing in progress...");
    }

    @Override
    public void dispenseProduct(VendingMachine machine) {
        Slot slot = machine.getSlot(machine.getSelectedSlot());
        Product product = slot.getProduct();

        // Dispense product
        slot.dispense();
        System.out.println("🎁 Dispensed: " + product.getName());

        // Return change
        double change = machine.getBalance() - product.getPrice();
        if (change > 0) {
            System.out.println("💰 Change returned: ₹" + change);
        }

        // Reset machine
        machine.resetBalance();
        machine.setSelectedSlot(-1);
        machine.setState(new IdleState());
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel during dispensing!");
    }
}
```

### Vending Machine (Context)

```java
public class VendingMachine {
    private VendingMachineState state;
    private Map<Integer, Slot> slots;
    private double balance;
    private int selectedSlot;

    public VendingMachine() {
        this.state = new IdleState();
        this.slots = new HashMap<>();
        this.balance = 0;
        this.selectedSlot = -1;
    }

    // ─── Delegate to current state ─────────────
    public void insertMoney(Coin coin) {
        state.insertMoney(this, coin);
    }

    public void selectProduct(int slotNumber) {
        state.selectProduct(this, slotNumber);
    }

    public void dispenseProduct() {
        state.dispenseProduct(this);
    }

    public void cancel() {
        state.cancel(this);
    }

    // ─── Admin operations ───────────────────────
    public void addSlot(int slotNumber, Product product, int quantity) {
        slots.put(slotNumber, new Slot(slotNumber, product, quantity));
    }

    public void restockSlot(int slotNumber, int count) {
        Slot slot = slots.get(slotNumber);
        if (slot != null) slot.restock(count);
    }

    public void displayProducts() {
        System.out.println("=== Available Products ===");
        for (Slot slot : slots.values()) {
            System.out.printf("Slot %d: %s - ₹%.0f (Qty: %d)%n",
                slot.getSlotNumber(), slot.getProduct().getName(),
                slot.getProduct().getPrice(), slot.getQuantity());
        }
    }

    // ─── Internal methods ───────────────────────
    public void addBalance(double amount) { this.balance += amount; }
    public void resetBalance() { this.balance = 0; }
    public double getBalance() { return balance; }
    public void setState(VendingMachineState state) { this.state = state; }
    public Slot getSlot(int number) { return slots.get(number); }
    public int getSelectedSlot() { return selectedSlot; }
    public void setSelectedSlot(int slot) { this.selectedSlot = slot; }
}
```

---

## 🧪 Usage Example

```java
VendingMachine machine = new VendingMachine();

// Admin setup
machine.addSlot(1, new Product("Coca Cola", 40), 5);
machine.addSlot(2, new Product("Pepsi", 35), 3);
machine.addSlot(3, new Product("Lays Chips", 20), 10);

// Display products
machine.displayProducts();

// User interaction
machine.insertMoney(Coin.FIFTY);       // → Inserted: ₹50 | Balance: ₹50
machine.selectProduct(1);               // → Dispensed: Coca Cola, Change: ₹10

// Another user
machine.insertMoney(Coin.TWENTY);      // → Inserted: ₹20 | Balance: ₹20
machine.selectProduct(1);               // → Dispensed: (insufficient, need ₹40)
machine.insertMoney(Coin.TWENTY);      // → Balance: ₹40
machine.selectProduct(1);               // → Dispensed: Coca Cola

// Cancel scenario
machine.insertMoney(Coin.HUNDRED);     // → Balance: ₹100
machine.cancel();                       // → Refund: ₹100
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **State** | IdleState, HasMoneyState, DispensingState |
| **Singleton** | VendingMachine can be singleton |
| **Strategy** | Can add different payment strategies |

---

## ⚠️ Edge Cases

- Product out of stock → allow selection of another product
- Insufficient money → prompt to insert more or cancel
- Machine cannot make change → rare, but should handle
- Power failure during dispensing → log transaction state
- Multiple coins inserted → accumulate balance

---

## 🔑 Key Takeaways

1. **State Pattern** is the star — each state defines valid operations
2. Invalid operations in a state → graceful error messages (not exceptions)
3. **Balance tracking** — accumulate coins, calculate change
4. Separate **admin operations** (restock) from **user operations** (buy)
5. **Dispensing is atomic** — cannot cancel mid-dispense
