# 12 - Splitwise / Expense Sharing System

## 📋 Problem Statement

Design an expense-sharing application that:
- Allows groups of users to split expenses
- Supports different split strategies (equal, exact, percentage)
- Tracks balances and simplifies debts

---

## 📌 Requirements

### Functional Requirements
1. **Add users** to the system
2. **Create groups** of users
3. **Add expenses** — paid by one or more users, split among participants
4. Split types: **Equal, Exact amount, Percentage**
5. **Track balances** — who owes whom how much
6. **Simplify debts** — minimize number of transactions
7. **Settle up** — record payment between users
8. **View balance sheet** per user or group

### Non-Functional Requirements
- Accurate to 2 decimal places
- Handle rounding errors in equal splits
- Thread-safe balance updates

---

## 🧩 Core Entities

```
User, Group, Expense, Split (Equal, Exact, Percent), 
Balance, Transaction, ExpenseService
```

---

## 📐 Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                      User                             │
│  - userId: String                                     │
│  - name: String                                       │
│  - email: String                                      │
│  - phone: String                                      │
│  - balanceMap: Map<String, Double>  (userId → amount) │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                    Expense                            │
│  - expenseId: String                                  │
│  - description: String                                │
│  - amount: double                                     │
│  - paidBy: User                                       │
│  - splits: List<Split>                                │
│  - group: Group (optional)                            │
│  - createdAt: LocalDateTime                           │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                  Split (abstract)                     │
│  - user: User                                         │
│  + getAmount(): double                                │
│  + validate(): boolean                                │
└───────────────┬──────────────────────────────────────┘
                │
     ┌──────────┼──────────┐
     │          │          │
 EqualSplit  ExactSplit  PercentSplit
```

---

## 💻 Code Implementation

### Split Classes (Strategy Pattern)

```java
public abstract class Split {
    protected User user;
    protected double amount;

    public Split(User user) {
        this.user = user;
    }

    public User getUser() { return user; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
}

public class EqualSplit extends Split {
    public EqualSplit(User user) {
        super(user);
    }
}

public class ExactSplit extends Split {
    public ExactSplit(User user, double amount) {
        super(user);
        this.amount = amount;
    }
}

public class PercentSplit extends Split {
    private double percent;

    public PercentSplit(User user, double percent) {
        super(user);
        this.percent = percent;
    }

    public double getPercent() { return percent; }
}
```

### Expense Class

```java
public class Expense {
    private String expenseId;
    private String description;
    private double amount;
    private User paidBy;
    private List<Split> splits;
    private SplitType splitType;
    private LocalDateTime createdAt;

    public Expense(String description, double amount, User paidBy, 
                   List<Split> splits, SplitType splitType) {
        this.expenseId = UUID.randomUUID().toString();
        this.description = description;
        this.amount = amount;
        this.paidBy = paidBy;
        this.splits = splits;
        this.splitType = splitType;
        this.createdAt = LocalDateTime.now();
    }

    // Getters
    public double getAmount() { return amount; }
    public User getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return splits; }
}

public enum SplitType {
    EQUAL, EXACT, PERCENT
}
```

### Expense Service (Core Logic)

```java
public class ExpenseService {
    private Map<String, User> users;
    private List<Expense> expenses;

    public ExpenseService() {
        this.users = new HashMap<>();
        this.expenses = new ArrayList<>();
    }

    public void addUser(User user) {
        users.put(user.getUserId(), user);
    }

    public Expense addExpense(String description, double amount, User paidBy,
                              List<Split> splits, SplitType splitType) {
        // Validate splits
        validateSplits(amount, splits, splitType);

        // Calculate split amounts
        calculateSplitAmounts(amount, splits, splitType);

        // Create expense
        Expense expense = new Expense(description, amount, paidBy, splits, splitType);
        expenses.add(expense);

        // Update balances
        updateBalances(paidBy, splits);

        return expense;
    }

    private void validateSplits(double amount, List<Split> splits, SplitType type) {
        switch (type) {
            case EXACT:
                double totalExact = splits.stream().mapToDouble(Split::getAmount).sum();
                if (Math.abs(totalExact - amount) > 0.01) {
                    throw new InvalidSplitException("Exact amounts don't add up to total");
                }
                break;

            case PERCENT:
                double totalPercent = splits.stream()
                    .map(s -> (PercentSplit) s)
                    .mapToDouble(PercentSplit::getPercent)
                    .sum();
                if (Math.abs(totalPercent - 100.0) > 0.01) {
                    throw new InvalidSplitException("Percentages don't add up to 100");
                }
                break;

            case EQUAL:
                // No validation needed
                break;
        }
    }

    private void calculateSplitAmounts(double amount, List<Split> splits, SplitType type) {
        switch (type) {
            case EQUAL:
                double perPerson = Math.round(amount / splits.size() * 100.0) / 100.0;
                double remainder = amount - (perPerson * splits.size());
                for (int i = 0; i < splits.size(); i++) {
                    double share = perPerson;
                    if (i == 0) share += remainder; // first person absorbs rounding diff
                    splits.get(i).setAmount(share);
                }
                break;

            case PERCENT:
                for (Split split : splits) {
                    PercentSplit ps = (PercentSplit) split;
                    ps.setAmount(Math.round(amount * ps.getPercent() / 100.0 * 100.0) / 100.0);
                }
                break;

            case EXACT:
                // Already set
                break;
        }
    }

    private void updateBalances(User paidBy, List<Split> splits) {
        for (Split split : splits) {
            User owes = split.getUser();
            double amount = split.getAmount();

            if (owes.getUserId().equals(paidBy.getUserId())) continue; // skip self

            // owes → paidBy: owes needs to pay paidBy
            // Update owes's balance: owes MORE to paidBy
            owes.updateBalance(paidBy.getUserId(), -amount);

            // Update paidBy's balance: paidBy is OWED more by owes
            paidBy.updateBalance(owes.getUserId(), amount);
        }
    }

    public void settleUp(User payer, User payee, double amount) {
        payer.updateBalance(payee.getUserId(), amount);
        payee.updateBalance(payer.getUserId(), -amount);
        System.out.println(payer.getName() + " paid ₹" + amount + " to " + payee.getName());
    }
}
```

### User Balance Management

```java
public class User {
    private String userId;
    private String name;
    private String email;
    private Map<String, Double> balanceMap; // userId → net balance

    public User(String userId, String name, String email) {
        this.userId = userId;
        this.name = name;
        this.email = email;
        this.balanceMap = new ConcurrentHashMap<>();
    }

    public void updateBalance(String otherUserId, double amount) {
        balanceMap.merge(otherUserId, amount, Double::sum);
        // Clean up zero balances
        if (Math.abs(balanceMap.getOrDefault(otherUserId, 0.0)) < 0.01) {
            balanceMap.remove(otherUserId);
        }
    }

    public void showBalances() {
        if (balanceMap.isEmpty()) {
            System.out.println(name + ": No balances");
            return;
        }
        for (Map.Entry<String, Double> entry : balanceMap.entrySet()) {
            double amount = entry.getValue();
            if (amount > 0) {
                System.out.println(name + " is owed ₹" + amount + " by " + entry.getKey());
            } else {
                System.out.println(name + " owes ₹" + Math.abs(amount) + " to " + entry.getKey());
            }
        }
    }

    // Getters
    public String getUserId() { return userId; }
    public String getName() { return name; }
    public Map<String, Double> getBalanceMap() { return balanceMap; }
}
```

### Debt Simplification (Greedy Algorithm)

```java
public class DebtSimplifier {
    
    /**
     * Minimize the number of transactions to settle all debts.
     * Uses greedy approach: match max creditor with max debtor.
     */
    public List<Transaction> simplifyDebts(List<User> users) {
        // Calculate net balance for each user
        Map<String, Double> netBalance = new HashMap<>();
        for (User user : users) {
            double net = user.getBalanceMap().values().stream()
                            .mapToDouble(Double::doubleValue).sum();
            if (Math.abs(net) > 0.01) {
                netBalance.put(user.getUserId(), net);
            }
        }

        // Separate into creditors (+ve) and debtors (-ve)
        PriorityQueue<double[]> creditors = new PriorityQueue<>((a, b) -> Double.compare(b[1], a[1]));
        PriorityQueue<double[]> debtors = new PriorityQueue<>((a, b) -> Double.compare(a[1], b[1]));

        for (Map.Entry<String, Double> entry : netBalance.entrySet()) {
            if (entry.getValue() > 0) {
                creditors.offer(new double[]{entry.getKey().hashCode(), entry.getValue()});
            } else {
                debtors.offer(new double[]{entry.getKey().hashCode(), entry.getValue()});
            }
        }

        List<Transaction> transactions = new ArrayList<>();

        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            double[] creditor = creditors.poll();
            double[] debtor = debtors.poll();

            double settleAmount = Math.min(creditor[1], Math.abs(debtor[1]));
            transactions.add(new Transaction(debtor[0], creditor[0], settleAmount));

            double creditorRemaining = creditor[1] - settleAmount;
            double debtorRemaining = debtor[1] + settleAmount;

            if (creditorRemaining > 0.01) creditors.offer(new double[]{creditor[0], creditorRemaining});
            if (debtorRemaining < -0.01) debtors.offer(new double[]{debtor[0], debtorRemaining});
        }

        return transactions;
    }
}
```

---

## 🧪 Usage Example

```java
ExpenseService service = new ExpenseService();

User alice = new User("U1", "Alice", "alice@email.com");
User bob = new User("U2", "Bob", "bob@email.com");
User charlie = new User("U3", "Charlie", "charlie@email.com");

service.addUser(alice);
service.addUser(bob);
service.addUser(charlie);

// Expense 1: Alice paid ₹900 dinner, split equally among 3
service.addExpense("Dinner", 900, alice,
    List.of(new EqualSplit(alice), new EqualSplit(bob), new EqualSplit(charlie)),
    SplitType.EQUAL);
// Bob owes Alice ₹300, Charlie owes Alice ₹300

// Expense 2: Bob paid ₹500 cab, exact split
service.addExpense("Cab", 500, bob,
    List.of(new ExactSplit(alice, 200), new ExactSplit(bob, 100), new ExactSplit(charlie, 200)),
    SplitType.EXACT);
// Alice owes Bob ₹200, Charlie owes Bob ₹200

// Show balances
alice.showBalances();
// Alice is owed ₹100 by Bob (300 - 200)
// Alice is owed ₹300 by Charlie

bob.showBalances();
// Bob owes ₹100 to Alice
// Bob is owed ₹200 by Charlie
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Split types (Equal, Exact, Percent) |
| **Observer** | Notifications when expense is added |
| **Facade** | ExpenseService hides complexity |

---

## ⚠️ Edge Cases

- Rounding errors in equal split → first person absorbs difference
- Percentage doesn't add to 100 → reject
- Exact amounts don't add to total → reject
- User not part of expense → exclude from split
- Self-payment → skip in balance update
- Zero-amount splits → ignore

---

## 🔑 Key Takeaways

1. **Balance Map** per user — `Map<userId, netAmount>` (positive = owed, negative = owes)
2. **Split Strategy** is the key abstraction — Equal, Exact, Percent
3. **Debt Simplification** uses greedy matching of max creditor with max debtor
4. **Rounding handling** is critical for financial accuracy
5. Balance is always **pair-wise** — if A owes B ₹100, B is owed ₹100 from A
