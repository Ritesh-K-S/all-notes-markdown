# 21 - Payment System

## 📋 Problem Statement

Design a payment processing system that:
- Supports multiple payment methods (Card, UPI, Wallet, Net Banking)
- Handles payment lifecycle (initiate, process, refund)
- Integrates with payment gateways

---

## 📌 Requirements

### Functional Requirements
1. Support **multiple payment methods**: Credit Card, Debit Card, UPI, Wallet, Net Banking
2. **Initiate payment** → Process → Success/Failure
3. **Refund** processing
4. **Transaction history** per user
5. **Retry** on failure
6. **Idempotency** — same payment request doesn't charge twice

---

## 💻 Code Implementation

### Payment Strategy Pattern

```java
public interface PaymentMethod {
    PaymentResult process(double amount, PaymentDetails details);
    PaymentResult refund(String transactionId, double amount);
    String getMethodName();
}

public class CreditCardPayment implements PaymentMethod {
    @Override
    public PaymentResult process(double amount, PaymentDetails details) {
        // Validate card, call payment gateway
        System.out.println("Processing credit card payment: ₹" + amount);
        return new PaymentResult(true, UUID.randomUUID().toString(), "Card payment successful");
    }

    @Override
    public PaymentResult refund(String transactionId, double amount) {
        System.out.println("Refunding ₹" + amount + " to credit card");
        return new PaymentResult(true, transactionId, "Refund successful");
    }

    @Override
    public String getMethodName() { return "CREDIT_CARD"; }
}

public class UPIPayment implements PaymentMethod {
    @Override
    public PaymentResult process(double amount, PaymentDetails details) {
        System.out.println("Processing UPI payment: ₹" + amount);
        return new PaymentResult(true, UUID.randomUUID().toString(), "UPI payment successful");
    }

    @Override
    public PaymentResult refund(String transactionId, double amount) {
        return new PaymentResult(true, transactionId, "UPI refund initiated");
    }

    @Override
    public String getMethodName() { return "UPI"; }
}

public class WalletPayment implements PaymentMethod {
    @Override
    public PaymentResult process(double amount, PaymentDetails details) {
        System.out.println("Processing wallet payment: ₹" + amount);
        return new PaymentResult(true, UUID.randomUUID().toString(), "Wallet payment successful");
    }

    @Override
    public PaymentResult refund(String transactionId, double amount) {
        return new PaymentResult(true, transactionId, "Wallet refund successful");
    }

    @Override
    public String getMethodName() { return "WALLET"; }
}
```

### Payment Factory

```java
public class PaymentMethodFactory {
    private static final Map<String, PaymentMethod> methods = Map.of(
        "CREDIT_CARD", new CreditCardPayment(),
        "DEBIT_CARD", new DebitCardPayment(),
        "UPI", new UPIPayment(),
        "WALLET", new WalletPayment(),
        "NET_BANKING", new NetBankingPayment()
    );

    public static PaymentMethod getPaymentMethod(String type) {
        PaymentMethod method = methods.get(type.toUpperCase());
        if (method == null) throw new UnsupportedPaymentException("Unknown method: " + type);
        return method;
    }
}
```

### Transaction and Payment Service

```java
public class Transaction {
    private String transactionId;
    private String orderId;
    private double amount;
    private String paymentMethodType;
    private TransactionStatus status;
    private String idempotencyKey;
    private LocalDateTime createdAt;
    private String failureReason;

    public Transaction(String orderId, double amount, String method, String idempotencyKey) {
        this.transactionId = UUID.randomUUID().toString();
        this.orderId = orderId;
        this.amount = amount;
        this.paymentMethodType = method;
        this.idempotencyKey = idempotencyKey;
        this.status = TransactionStatus.INITIATED;
        this.createdAt = LocalDateTime.now();
    }
}

public enum TransactionStatus {
    INITIATED, PROCESSING, SUCCESS, FAILED, REFUND_INITIATED, REFUNDED
}

public class PaymentService {
    private Map<String, Transaction> transactions;
    private Set<String> processedIdempotencyKeys;
    private static final int MAX_RETRIES = 3;

    public PaymentService() {
        this.transactions = new ConcurrentHashMap<>();
        this.processedIdempotencyKeys = ConcurrentHashMap.newKeySet();
    }

    public Transaction initiatePayment(String orderId, double amount, 
                                        String methodType, String idempotencyKey) {
        // Idempotency check
        if (processedIdempotencyKeys.contains(idempotencyKey)) {
            return findByIdempotencyKey(idempotencyKey);
        }

        PaymentMethod method = PaymentMethodFactory.getPaymentMethod(methodType);
        Transaction txn = new Transaction(orderId, amount, methodType, idempotencyKey);
        transactions.put(txn.getTransactionId(), txn);

        // Process with retry
        txn.setStatus(TransactionStatus.PROCESSING);
        PaymentResult result = processWithRetry(method, amount, MAX_RETRIES);

        if (result.isSuccess()) {
            txn.setStatus(TransactionStatus.SUCCESS);
        } else {
            txn.setStatus(TransactionStatus.FAILED);
            txn.setFailureReason(result.getMessage());
        }

        processedIdempotencyKeys.add(idempotencyKey);
        return txn;
    }

    public Transaction refund(String transactionId) {
        Transaction original = transactions.get(transactionId);
        if (original == null || original.getStatus() != TransactionStatus.SUCCESS) {
            throw new InvalidRefundException("Cannot refund this transaction");
        }

        PaymentMethod method = PaymentMethodFactory.getPaymentMethod(original.getPaymentMethodType());
        PaymentResult result = method.refund(transactionId, original.getAmount());

        original.setStatus(result.isSuccess() ? TransactionStatus.REFUNDED : TransactionStatus.FAILED);
        return original;
    }

    private PaymentResult processWithRetry(PaymentMethod method, double amount, int maxRetries) {
        PaymentResult result = null;
        for (int i = 0; i < maxRetries; i++) {
            result = method.process(amount, null);
            if (result.isSuccess()) return result;
            System.out.println("Retry " + (i + 1) + "/" + maxRetries);
        }
        return result;
    }
}
```

---

## 🧪 Usage Example

```java
PaymentService service = new PaymentService();

// Pay for order
Transaction txn = service.initiatePayment("ORD-001", 999.0, "UPI", "idem-key-001");
// → UPI payment successful, status = SUCCESS

// Duplicate request (same idempotency key)
Transaction dup = service.initiatePayment("ORD-001", 999.0, "UPI", "idem-key-001");
// → Returns existing transaction (no double charge)

// Refund
Transaction refund = service.refund(txn.getTransactionId());
// → Refund initiated
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Payment methods — Card, UPI, Wallet |
| **Factory** | PaymentMethodFactory creates processors |
| **Idempotency** | Duplicate request detection via key |

---

## 🔑 Key Takeaways

1. **Strategy Pattern** — each payment method implements the same interface
2. **Factory** creates the right payment processor
3. **Idempotency key** prevents double charging
4. **Retry with backoff** for transient failures
5. **Transaction state machine**: INITIATED → PROCESSING → SUCCESS/FAILED → REFUNDED
