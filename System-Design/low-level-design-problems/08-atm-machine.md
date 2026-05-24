# 08 - ATM Machine

## 📋 Problem Statement

Design an ATM system that:
- Authenticates users via card and PIN
- Supports cash withdrawal, deposit, balance inquiry, and transfer
- Dispenses cash using optimal denomination selection

---

## 📌 Requirements

### Functional Requirements
1. **Card insertion** and **PIN authentication**
2. **Balance inquiry**
3. **Cash withdrawal** — dispense in optimal denominations
4. **Cash deposit**
5. **Fund transfer** to another account
6. ATM has **limited cash** in different denominations (2000, 500, 200, 100)
7. **Transaction receipt** generation
8. **Session timeout** after inactivity

### Non-Functional Requirements
- Thread-safe (one user at a time per ATM)
- Secure PIN handling
- Transaction logging/audit trail

---

## 🧩 Core Entities

```
ATM, Card, Account, Bank, Transaction, CashDispenser, 
Screen, Keypad, Printer, ATMState
```

---

## 📐 Class Diagram

```
┌────────────────────────────────────────────────────────┐
│                      ATM                                │
│  - atmId: String                                        │
│  - state: ATMState         ← State Pattern              │
│  - cashDispenser: CashDispenser                         │
│  - screen: Screen                                       │
│  - keypad: Keypad                                       │
│  - printer: Printer                                     │
│  - currentCard: Card                                    │
│  - currentAccount: Account                              │
│  + insertCard(card): void                               │
│  + authenticatePin(pin): boolean                        │
│  + selectTransaction(type): void                        │
│  + withdraw(amount): void                               │
│  + deposit(amount): void                                │
│  + checkBalance(): double                               │
│  + ejectCard(): void                                    │
└────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────────────────┐
│   ATMState        │     │    CashDispenser              │
│  ─ ─ ─ ─ ─ ─     │     │  - denominations: Map         │
│  IDLE             │     │  - chain: DispenseChain       │
│  CARD_INSERTED    │     │  + dispense(amount): boolean  │
│  AUTHENTICATED    │     │  + canDispense(amount): bool  │
│  TRANSACTION      │     │  + addCash(denom, count)      │
│  ERROR            │     └──────────────────────────────┘
└──────────────────┘

┌──────────────────────────────────────────────────────────┐
│                     Account                               │
│  - accountNumber: String                                  │
│  - balance: double                                        │
│  - pin: String (hashed)                                   │
│  - card: Card                                             │
│  + withdraw(amount): boolean                              │
│  + deposit(amount): void                                  │
│  + getBalance(): double                                   │
│  + validatePin(pin): boolean                              │
└──────────────────────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum ATMState {
    IDLE, CARD_INSERTED, AUTHENTICATED, TRANSACTION, ERROR
}

public enum TransactionType {
    WITHDRAWAL, DEPOSIT, BALANCE_INQUIRY, TRANSFER
}
```

---

## 💻 Code Implementation

### Cash Dispenser — Chain of Responsibility Pattern

```java
public abstract class CashHandler {
    protected CashHandler nextHandler;
    protected int denomination;
    protected int count;

    public CashHandler(int denomination, int count) {
        this.denomination = denomination;
        this.count = count;
    }

    public void setNextHandler(CashHandler next) {
        this.nextHandler = next;
    }

    public void dispense(int amount) {
        if (amount >= denomination) {
            int needed = amount / denomination;
            int used = Math.min(needed, count);
            int remaining = amount - (used * denomination);
            count -= used;

            System.out.println("Dispensing " + used + " × ₹" + denomination);

            if (remaining > 0 && nextHandler != null) {
                nextHandler.dispense(remaining);
            } else if (remaining > 0) {
                throw new InsufficientDenominationException("Cannot dispense remaining: ₹" + remaining);
            }
        } else if (nextHandler != null) {
            nextHandler.dispense(amount);
        }
    }
}

public class TwoThousandHandler extends CashHandler {
    public TwoThousandHandler(int count) { super(2000, count); }
}

public class FiveHundredHandler extends CashHandler {
    public FiveHundredHandler(int count) { super(500, count); }
}

public class TwoHundredHandler extends CashHandler {
    public TwoHundredHandler(int count) { super(200, count); }
}

public class HundredHandler extends CashHandler {
    public HundredHandler(int count) { super(100, count); }
}
```

### CashDispenser (Facade)

```java
public class CashDispenser {
    private CashHandler chain;
    private Map<Integer, Integer> denominations; // denomination → count

    public CashDispenser() {
        denominations = new LinkedHashMap<>();
        denominations.put(2000, 10);
        denominations.put(500, 20);
        denominations.put(200, 20);
        denominations.put(100, 50);
        buildChain();
    }

    private void buildChain() {
        CashHandler h2000 = new TwoThousandHandler(denominations.get(2000));
        CashHandler h500 = new FiveHundredHandler(denominations.get(500));
        CashHandler h200 = new TwoHundredHandler(denominations.get(200));
        CashHandler h100 = new HundredHandler(denominations.get(100));

        h2000.setNextHandler(h500);
        h500.setNextHandler(h200);
        h200.setNextHandler(h100);

        this.chain = h2000;
    }

    public boolean canDispense(int amount) {
        if (amount % 100 != 0) return false;
        int totalCash = denominations.entrySet().stream()
                            .mapToInt(e -> e.getKey() * e.getValue())
                            .sum();
        return amount <= totalCash;
    }

    public void dispense(int amount) {
        if (!canDispense(amount)) {
            throw new InsufficientCashException("ATM doesn't have enough cash");
        }
        chain.dispense(amount);
    }
}
```

### ATM State Machine (State Pattern)

```java
public interface ATMStateHandler {
    void insertCard(ATM atm, Card card);
    void authenticatePin(ATM atm, String pin);
    void selectTransaction(ATM atm, TransactionType type);
    void withdraw(ATM atm, int amount);
    void deposit(ATM atm, int amount);
    void checkBalance(ATM atm);
    void ejectCard(ATM atm);
}

// ─── IDLE STATE ─────────────────────────────
public class IdleState implements ATMStateHandler {
    @Override
    public void insertCard(ATM atm, Card card) {
        atm.setCurrentCard(card);
        atm.setState(ATMState.CARD_INSERTED);
        System.out.println("Card inserted. Enter PIN.");
    }

    @Override
    public void authenticatePin(ATM atm, String pin) {
        throw new IllegalStateException("Insert card first");
    }

    // ... other methods throw IllegalStateException
}

// ─── CARD INSERTED STATE ────────────────────
public class CardInsertedState implements ATMStateHandler {
    @Override
    public void authenticatePin(ATM atm, String pin) {
        Account account = BankService.getAccount(atm.getCurrentCard());
        if (account.validatePin(pin)) {
            atm.setCurrentAccount(account);
            atm.setState(ATMState.AUTHENTICATED);
            System.out.println("PIN verified. Select transaction.");
        } else {
            System.out.println("Invalid PIN!");
            atm.ejectCard();
        }
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.setCurrentCard(null);
        atm.setState(ATMState.IDLE);
        System.out.println("Card ejected.");
    }

    // ... other methods throw IllegalStateException
}

// ─── AUTHENTICATED STATE ────────────────────
public class AuthenticatedState implements ATMStateHandler {
    @Override
    public void withdraw(ATM atm, int amount) {
        Account account = atm.getCurrentAccount();
        if (account.getBalance() < amount) {
            System.out.println("Insufficient balance!");
            return;
        }
        if (!atm.getCashDispenser().canDispense(amount)) {
            System.out.println("ATM cannot dispense this amount!");
            return;
        }

        account.withdraw(amount);
        atm.getCashDispenser().dispense(amount);
        System.out.println("Withdrawal successful. Remaining: ₹" + account.getBalance());
    }

    @Override
    public void checkBalance(ATM atm) {
        System.out.println("Balance: ₹" + atm.getCurrentAccount().getBalance());
    }

    @Override
    public void deposit(ATM atm, int amount) {
        atm.getCurrentAccount().deposit(amount);
        System.out.println("Deposited ₹" + amount + ". New Balance: ₹" + atm.getCurrentAccount().getBalance());
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.setCurrentCard(null);
        atm.setCurrentAccount(null);
        atm.setState(ATMState.IDLE);
        System.out.println("Card ejected. Thank you!");
    }
}
```

### ATM Class

```java
public class ATM {
    private String atmId;
    private ATMState state;
    private Map<ATMState, ATMStateHandler> stateHandlers;
    private CashDispenser cashDispenser;
    private Card currentCard;
    private Account currentAccount;

    public ATM(String atmId) {
        this.atmId = atmId;
        this.state = ATMState.IDLE;
        this.cashDispenser = new CashDispenser();
        this.stateHandlers = Map.of(
            ATMState.IDLE, new IdleState(),
            ATMState.CARD_INSERTED, new CardInsertedState(),
            ATMState.AUTHENTICATED, new AuthenticatedState()
        );
    }

    public void insertCard(Card card) {
        stateHandlers.get(state).insertCard(this, card);
    }

    public void authenticatePin(String pin) {
        stateHandlers.get(state).authenticatePin(this, pin);
    }

    public void withdraw(int amount) {
        stateHandlers.get(state).withdraw(this, amount);
    }

    public void checkBalance() {
        stateHandlers.get(state).checkBalance(this);
    }

    public void deposit(int amount) {
        stateHandlers.get(state).deposit(this, amount);
    }

    public void ejectCard() {
        stateHandlers.get(state).ejectCard(this);
    }

    // Getters and setters
    public void setState(ATMState state) { this.state = state; }
    public Card getCurrentCard() { return currentCard; }
    public void setCurrentCard(Card card) { this.currentCard = card; }
    public Account getCurrentAccount() { return currentAccount; }
    public void setCurrentAccount(Account account) { this.currentAccount = account; }
    public CashDispenser getCashDispenser() { return cashDispenser; }
}
```

---

## 🧪 Usage Example

```java
ATM atm = new ATM("ATM-001");
Card card = new Card("1234-5678-9012", "Ritesh");

atm.insertCard(card);           // → Card inserted. Enter PIN.
atm.authenticatePin("1234");    // → PIN verified. Select transaction.
atm.checkBalance();             // → Balance: ₹50000
atm.withdraw(3500);             // → Dispensing 1 × ₹2000, 1 × ₹500, 5 × ₹200
atm.ejectCard();                // → Card ejected. Thank you!
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **State** | ATM states: IDLE → CARD_INSERTED → AUTHENTICATED |
| **Chain of Responsibility** | Cash dispensing: 2000 → 500 → 200 → 100 |
| **Singleton** | ATM instance per machine |
| **Facade** | CashDispenser hides chain complexity |

---

## ⚠️ Edge Cases

- 3 wrong PIN attempts → block card
- Amount not multiple of 100 → reject
- ATM out of specific denomination → use next denomination
- ATM completely out of cash → display "Out of Service"
- Card stuck / power failure → maintain session state
- Concurrent access → one user per ATM enforced

---

## 🔑 Key Takeaways

1. **State Pattern** is the primary pattern — ATM behavior changes based on state
2. **Chain of Responsibility** for denomination-based dispensing
3. Each state only allows **valid transitions**
4. Cash dispenser handles greedy denomination selection
5. Security: PIN should be **hashed**, not stored as plaintext
