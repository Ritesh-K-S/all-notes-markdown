# How Stripe/Razorpay Handles Payment Processing

> **What you'll learn**: How payment platforms process billions of dollars in transactions with exactly-once guarantees, prevent fraud in real-time, handle multi-currency settlements, and maintain the highest reliability standards in the industry — because a payment bug means real money disappearing.

---

## Real-Life Analogy

Imagine you're running a **global bank's transfer desk**, but with insane requirements:

- You handle **millions of money transfers per day** across every currency on Earth
- Every transfer MUST happen **exactly once** — if you charge someone twice, that's a lawsuit. If you don't charge at all, the merchant loses money.
- You have **0.1 seconds** to decide if a transaction is fraudulent (while a thief's stolen credit card is being swiped)
- Multiple parties are involved: the buyer, the seller, the buyer's bank, the seller's bank, card networks (Visa/Mastercard), and YOU in the middle
- If your system goes down for even **5 minutes**, thousands of businesses worldwide can't accept payments — they lose sales, you lose trust
- You must comply with hundreds of regulations across 40+ countries

This is why payments engineering is considered one of the **hardest problems in software** — the cost of bugs is not broken features, it's lost money.

---

## Core Concept Explained Step-by-Step

### Step 1: The Payment Flow — Who's Involved?

```
┌─────────────────────────────────────────────────────────────────────┐
│              THE PAYMENT ECOSYSTEM                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Customer ──▶ Merchant ──▶ Payment Gateway ──▶ Payment Processor    │
│  (Buyer)     (Seller)     (Stripe/Razorpay)  ──▶ Card Network      │
│                                               ──▶ Issuing Bank      │
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐                  │
│  │ Customer │    │ Merchant │    │ Stripe /     │                  │
│  │          │    │ (Amazon, │    │ Razorpay     │                  │
│  │ Has a    │    │  Swiggy, │    │              │                  │
│  │ credit   │    │  etc.)   │    │ Payment      │                  │
│  │ card     │    │          │    │ Gateway      │                  │
│  └────┬─────┘    └────┬─────┘    └──────┬───────┘                  │
│       │               │                  │                           │
│       │  Enters card  │  API call:       │                           │
│       │  details      │  charge(amount)  │                           │
│       └───────────────┘                  │                           │
│                                          │                           │
│                               ┌──────────┼──────────┐               │
│                               ▼          ▼          ▼               │
│                        ┌──────────┐ ┌─────────┐ ┌─────────┐        │
│                        │  Card    │ │Acquiring│ │ Issuing │        │
│                        │  Network │ │  Bank   │ │  Bank   │        │
│                        │(Visa/MC) │ │(Merchant│ │(Customer│        │
│                        │          │ │  bank)  │ │  bank)  │        │
│                        └──────────┘ └─────────┘ └─────────┘        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 2: What Happens When You Pay Online

```
Customer clicks "Pay ₹999" on a merchant's website
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: TOKENIZATION (Client-side)                                  │
│ • Card number entered in Stripe/Razorpay's embedded form           │
│ • Card details NEVER touch merchant's server (PCI compliance)      │
│ • Converted to a one-time token: "tok_1234abcd"                   │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 2: CREATE CHARGE (Merchant → Stripe API)                      │
│ • Merchant sends: {token: "tok_1234", amount: 999, currency: "INR"}│
│ • Stripe validates: Is token valid? Is merchant authorized?        │
│ • Idempotency key attached (prevents duplicate charges)            │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 3: FRAUD CHECK (< 100ms)                                      │
│ • ML model scores transaction risk (0-100)                         │
│ • Checks: Unusual amount? New device? IP from different country?  │
│ • High risk → decline or request 3D Secure (OTP)                  │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 4: AUTHORIZATION (Stripe → Card Network → Issuing Bank)       │
│ • "Does this card have ₹999 available?"                           │
│ • Issuing bank checks balance/credit limit                         │
│ • Returns: APPROVED or DECLINED                                    │
│ • If approved: Amount is HELD (not yet transferred)               │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 5: CAPTURE (happens immediately or later)                     │
│ • Converts authorization hold into actual charge                   │
│ • Money moves: Customer's bank → Card Network → Acquiring Bank    │
│                → Stripe/Razorpay (takes fee) → Merchant            │
│ • Settlement usually takes 1-3 business days                       │
└────────────────────────────────────────────────────────────────────┘
```

### Step 3: The Double-Entry Ledger

The most critical component — tracking every rupee:

```
┌─────────────────────────────────────────────────────────────────┐
│              DOUBLE-ENTRY ACCOUNTING LEDGER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  RULE: Every transaction has TWO entries that MUST balance        │
│  Debit = Credit (always, no exceptions)                          │
│                                                                   │
│  Example: Customer pays ₹1000, Stripe takes 2% fee              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Transaction ID: txn_abc123                             │     │
│  │  Created: 2024-01-15 10:30:45 UTC                      │     │
│  │                                                        │     │
│  │  Entry 1: DEBIT  customer_funds     ₹1000 (money out) │     │
│  │  Entry 2: CREDIT merchant_balance   ₹980  (money in)  │     │
│  │  Entry 3: CREDIT stripe_revenue     ₹20   (fee)       │     │
│  │                                                        │     │
│  │  Sum of debits (₹1000) = Sum of credits (₹980 + ₹20) ✓ │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                   │
│  Properties:                                                      │
│  • IMMUTABLE: Entries are NEVER modified or deleted              │
│  • Corrections are done by adding REVERSAL entries               │
│  • Full audit trail of every rupee ever moved                    │
│  • Can reconstruct any account balance at any point in time      │
│                                                                   │
│  For refund:                                                      │
│  Entry 1: DEBIT  merchant_balance   ₹980  (take back from merchant)│
│  Entry 2: DEBIT  stripe_revenue     ₹20   (return our fee)      │
│  Entry 3: CREDIT customer_funds     ₹1000 (return to customer)  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 4: Idempotency — Preventing Double Charges

```
┌─────────────────────────────────────────────────────────────────┐
│              IDEMPOTENCY IN PAYMENTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Problem: Network timeout. Did the charge go through or not?     │
│                                                                   │
│  Merchant ──── charge(₹999) ────▶ Stripe                        │
│           ◀── timeout (no response) ──×                          │
│                                                                   │
│  What now? Retry? But what if it DID succeed?                    │
│  Retrying could charge the customer TWICE! (₹1998)               │
│                                                                   │
│  SOLUTION: Idempotency Keys                                      │
│                                                                   │
│  Request 1: POST /charges                                        │
│             Idempotency-Key: "order_12345_attempt_1"             │
│             {amount: 999, token: "tok_abc"}                       │
│             → Server processes, saves result                      │
│             → Response lost (network timeout)                     │
│                                                                   │
│  Request 2: POST /charges (RETRY - same idempotency key)        │
│             Idempotency-Key: "order_12345_attempt_1" (SAME!)     │
│             {amount: 999, token: "tok_abc"}                       │
│             → Server sees: "I already processed this!"            │
│             → Returns the SAME result as before                   │
│             → NO duplicate charge                                 │
│                                                                   │
│  Implementation:                                                  │
│  ┌────────────────────────────────────────────────────┐          │
│  │  idempotency_keys table:                           │          │
│  │  ┌─────────────────────┬────────────┬───────────┐ │          │
│  │  │ key                 │ status     │ response  │ │          │
│  │  ├─────────────────────┼────────────┼───────────┤ │          │
│  │  │ order_12345_attempt │ completed  │ {charge}  │ │          │
│  │  └─────────────────────┴────────────┴───────────┘ │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Payment Processing Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│            STRIPE/RAZORPAY INTERNAL ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    API LAYER                               │      │
│  │  • Rate limiting   • Authentication  • Input validation  │      │
│  │  • Idempotency checking                                  │      │
│  └──────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                 ORCHESTRATION LAYER                        │      │
│  │  Payment State Machine: Manages the lifecycle             │      │
│  │  pending → processing → authorized → captured → settled   │      │
│  └─────────────┬────────────┬────────────┬──────────────────┘      │
│                │            │            │                           │
│       ┌────────┘     ┌──────┘     ┌──────┘                         │
│       ▼              ▼            ▼                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐                     │
│  │  Fraud   │  │  Payment │  │   Ledger     │                     │
│  │  Engine  │  │  Router  │  │   Service    │                     │
│  │          │  │          │  │              │                     │
│  │ ML model │  │ Routes   │  │ Double-entry │                     │
│  │ 100ms    │  │ to right │  │ accounting   │                     │
│  │ decision │  │ processor│  │ Immutable    │                     │
│  └──────────┘  └────┬─────┘  └──────────────┘                     │
│                      │                                              │
│         ┌────────────┼────────────────┐                             │
│         ▼            ▼                ▼                             │
│  ┌──────────┐  ┌──────────┐    ┌──────────┐                       │
│  │   Visa   │  │Mastercard│    │   UPI    │                       │
│  │  Gateway │  │ Gateway  │    │  Gateway │                       │
│  └──────────┘  └──────────┘    └──────────┘                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                 SETTLEMENT ENGINE (Async)                  │      │
│  │  • Batch processes settled transactions daily             │      │
│  │  • Calculates net amounts per merchant                    │      │
│  │  • Initiates bank transfers (T+1 to T+3)                 │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Payment State Machine

Every payment goes through a strict state machine:

```
┌─────────────────────────────────────────────────────────────────┐
│              PAYMENT STATE MACHINE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────┐    ┌────────────┐    ┌────────────┐                │
│  │ CREATED │───▶│ PROCESSING │───▶│ AUTHORIZED │                │
│  └─────────┘    └────────────┘    └──────┬─────┘                │
│       │                                   │                       │
│       │                          ┌────────┴────────┐              │
│       │                          ▼                 ▼              │
│       │                   ┌──────────┐      ┌──────────┐         │
│       │                   │ CAPTURED │      │  VOIDED  │         │
│       │                   └────┬─────┘      └──────────┘         │
│       │                        │                                  │
│       │                        ▼                                  │
│       │                   ┌──────────┐                            │
│       │                   │ SETTLED  │ (money transferred)        │
│       │                   └────┬─────┘                            │
│       │                        │                                  │
│       │                        ▼                                  │
│       │                   ┌──────────┐                            │
│       │                   │ REFUNDED │ (partial or full)          │
│       │                   └──────────┘                            │
│       │                                                           │
│       └──────────────────▶ ┌──────────┐                          │
│                            │  FAILED  │ (declined, error)         │
│                            └──────────┘                          │
│                                                                   │
│  Key rules:                                                       │
│  • Each transition is ATOMIC (all or nothing)                    │
│  • Each transition is LOGGED (full audit trail)                  │
│  • No skipping states (can't go from CREATED → SETTLED)         │
│  • IDEMPOTENT: Same request in same state → same result          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Fraud Detection System

```
┌─────────────────────────────────────────────────────────────────┐
│              REAL-TIME FRAUD DETECTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Decision required: APPROVE or DECLINE                           │
│  Time budget: < 100ms (user is waiting)                          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  INPUT SIGNALS:                                        │      │
│  │                                                       │      │
│  │  • Card details (BIN, country, card type)            │      │
│  │  • Transaction amount & currency                      │      │
│  │  • IP address & geolocation                           │      │
│  │  • Device fingerprint                                 │      │
│  │  • Velocity (how many transactions this hour?)       │      │
│  │  • Merchant risk category                            │      │
│  │  • Historical patterns for this card                 │      │
│  │  • Time of day / day of week                         │      │
│  └───────────────────────────────────────────────────────┘      │
│                          │                                        │
│                          ▼                                        │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  ML SCORING ENGINE:                                    │      │
│  │                                                       │      │
│  │  Rule 1: Card country ≠ IP country     → +30 risk    │      │
│  │  Rule 2: Amount > 5x average for card  → +25 risk    │      │
│  │  Rule 3: New device + large amount     → +20 risk    │      │
│  │  Rule 4: Known good customer pattern   → -40 risk    │      │
│  │  Rule 5: 10 transactions in 1 minute   → +50 risk    │      │
│  │  ...                                                  │      │
│  │  Neural network score:                  → 0-100      │      │
│  └───────────────────────────────────────────────────────┘      │
│                          │                                        │
│                          ▼                                        │
│  ┌───────────────────────────────────────────────────────┐      │
│  │  DECISION:                                             │      │
│  │                                                       │      │
│  │  Score 0-30:   APPROVE (low risk)                    │      │
│  │  Score 30-70:  Request 3D Secure/OTP (medium risk)   │      │
│  │  Score 70-100: DECLINE (high risk)                   │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Currency & Cross-Border Payments

```
┌─────────────────────────────────────────────────────────────────┐
│         CROSS-BORDER PAYMENT FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Indian customer buys from US merchant in USD                    │
│                                                                   │
│  Customer (India)              Stripe              Merchant (US) │
│  Card: ₹ INR                                     Wants: $ USD   │
│       │                          │                     │         │
│       │─── Pay $50 ─────────────▶│                     │         │
│       │                          │                     │         │
│       │                          │  1. Convert:        │         │
│       │                          │     $50 × 83.5 = ₹4175       │
│       │                          │                     │         │
│       │                          │  2. Charge Indian   │         │
│       │◀── ₹4175 charged ────────│     card in INR     │         │
│       │                          │                     │         │
│       │                          │  3. Settle with     │         │
│       │                          │     merchant in USD │         │
│       │                          │─── $50 paid ───────▶│         │
│       │                          │   (minus 2.9% fee)  │         │
│       │                          │   = $48.55          │         │
│                                                                   │
│  Stripe handles:                                                  │
│  • Real-time FX rate lookup                                      │
│  • FX markup (typically 1-2%)                                    │
│  • Multi-currency settlement                                     │
│  • Regulatory compliance (RBI rules for India, etc.)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Idempotent Payment Processing

```python
# Idempotent payment processing - the most critical pattern in payments
import uuid
import hashlib
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Dict
from datetime import datetime

class PaymentStatus(Enum):
    CREATED = "created"
    PROCESSING = "processing"
    AUTHORIZED = "authorized"
    CAPTURED = "captured"
    FAILED = "failed"
    REFUNDED = "refunded"

@dataclass
class Payment:
    payment_id: str
    amount: int          # In smallest unit (paise/cents)
    currency: str
    status: PaymentStatus
    idempotency_key: str
    created_at: datetime
    metadata: Dict = field(default_factory=dict)

class PaymentService:
    """
    Demonstrates idempotent payment processing.
    The same request with the same idempotency key NEVER creates 
    a duplicate charge — the most critical invariant in payments.
    """
    
    def __init__(self):
        self.payments: Dict[str, Payment] = {}
        self.idempotency_store: Dict[str, str] = {}  # key → payment_id
    
    def create_payment(self, amount: int, currency: str, 
                       token: str, idempotency_key: str) -> Payment:
        """
        Create a payment charge. IDEMPOTENT:
        Same idempotency_key → returns same result (no duplicate charge).
        """
        
        # CHECK: Have we seen this idempotency key before?
        if idempotency_key in self.idempotency_store:
            existing_id = self.idempotency_store[idempotency_key]
            return self.payments[existing_id]  # Return SAME result
        
        # NEW payment: process it
        payment_id = f"pay_{uuid.uuid4().hex[:16]}"
        
        payment = Payment(
            payment_id=payment_id,
            amount=amount,
            currency=currency,
            status=PaymentStatus.CREATED,
            idempotency_key=idempotency_key,
            created_at=datetime.utcnow()
        )
        
        # CRITICAL: Store idempotency key BEFORE processing
        # (prevents race conditions with retries)
        self.idempotency_store[idempotency_key] = payment_id
        self.payments[payment_id] = payment
        
        # Process: Authorize with card network
        try:
            payment.status = PaymentStatus.PROCESSING
            authorized = self._authorize_with_network(token, amount, currency)
            
            if authorized:
                payment.status = PaymentStatus.AUTHORIZED
                self._capture(payment)  # Auto-capture
                payment.status = PaymentStatus.CAPTURED
            else:
                payment.status = PaymentStatus.FAILED
        except Exception as e:
            payment.status = PaymentStatus.FAILED
            payment.metadata["error"] = str(e)
        
        return payment
    
    def _authorize_with_network(self, token, amount, currency) -> bool:
        """Call Visa/Mastercard network for authorization."""
        # In production: HTTP call to card network with timeout + retry
        return True  # Simplified
    
    def _capture(self, payment: Payment):
        """Convert auth hold to actual charge. Write to ledger."""
        # In production: write double-entry ledger records
        self._write_ledger_entry(
            debit_account=f"customer_{payment.payment_id}",
            credit_account="merchant_balance",
            amount=payment.amount
        )
    
    def _write_ledger_entry(self, debit_account, credit_account, amount):
        """Immutable ledger entry (append-only)."""
        pass  # In production: write to append-only ledger table

# Usage showing idempotency
service = PaymentService()
key = "order_789_attempt_1"

# First call: processes payment
result1 = service.create_payment(99900, "INR", "tok_abc", key)
print(f"First: {result1.payment_id}, {result1.status}")

# Retry (same key): returns SAME result, NO duplicate charge!
result2 = service.create_payment(99900, "INR", "tok_abc", key)
print(f"Retry: {result2.payment_id}, {result2.status}")
assert result1.payment_id == result2.payment_id  # Same payment!
```

### Java — Double-Entry Ledger

```java
import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentLinkedQueue;

/**
 * Double-entry accounting ledger — the financial heart of any payment system.
 * Every money movement creates balanced debit + credit entries.
 * Entries are IMMUTABLE (append-only) for audit compliance.
 */
public class PaymentLedger {
    // Append-only log of all financial entries
    private final ConcurrentLinkedQueue<LedgerEntry> entries = 
        new ConcurrentLinkedQueue<>();
    
    // Account balances (computed from entries)
    private final Map<String, BigDecimal> balances = new HashMap<>();

    public String recordPayment(String paymentId, String customerId, 
                                 String merchantId, BigDecimal amount, 
                                 BigDecimal platformFee) {
        String txnId = UUID.randomUUID().toString();
        BigDecimal merchantAmount = amount.subtract(platformFee);
        
        // ATOMIC: All entries for this transaction succeed or fail together
        List<LedgerEntry> batch = List.of(
            new LedgerEntry(txnId, customerId, EntryType.DEBIT, amount,
                "Payment for order"),
            new LedgerEntry(txnId, merchantId, EntryType.CREDIT, merchantAmount,
                "Sale revenue"),
            new LedgerEntry(txnId, "platform_revenue", EntryType.CREDIT, platformFee,
                "Platform fee (2%)")
        );
        
        // Validate: debits MUST equal credits
        BigDecimal totalDebits = batch.stream()
            .filter(e -> e.type == EntryType.DEBIT)
            .map(e -> e.amount).reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalCredits = batch.stream()
            .filter(e -> e.type == EntryType.CREDIT)
            .map(e -> e.amount).reduce(BigDecimal.ZERO, BigDecimal::add);
        
        if (totalDebits.compareTo(totalCredits) != 0) {
            throw new IllegalStateException(
                "CRITICAL: Debits ≠ Credits! This would corrupt the ledger.");
        }
        
        // Persist (in production: single DB transaction)
        batch.forEach(entry -> {
            entries.add(entry);
            updateBalance(entry);
        });
        
        return txnId;
    }

    public String recordRefund(String originalTxnId, String customerId, 
                                String merchantId, BigDecimal amount,
                                BigDecimal platformFee) {
        // Refund = REVERSE entries (debit becomes credit and vice versa)
        String txnId = "refund_" + UUID.randomUUID().toString();
        BigDecimal merchantAmount = amount.subtract(platformFee);
        
        List<LedgerEntry> batch = List.of(
            new LedgerEntry(txnId, customerId, EntryType.CREDIT, amount,
                "Refund to customer"),
            new LedgerEntry(txnId, merchantId, EntryType.DEBIT, merchantAmount,
                "Refund deducted from merchant"),
            new LedgerEntry(txnId, "platform_revenue", EntryType.DEBIT, platformFee,
                "Fee returned on refund")
        );
        
        batch.forEach(entry -> { entries.add(entry); updateBalance(entry); });
        return txnId;
    }

    private void updateBalance(LedgerEntry entry) {
        BigDecimal current = balances.getOrDefault(entry.accountId, BigDecimal.ZERO);
        if (entry.type == EntryType.CREDIT) {
            balances.put(entry.accountId, current.add(entry.amount));
        } else {
            balances.put(entry.accountId, current.subtract(entry.amount));
        }
    }

    public BigDecimal getBalance(String accountId) {
        return balances.getOrDefault(accountId, BigDecimal.ZERO);
    }

    enum EntryType { DEBIT, CREDIT }
    
    record LedgerEntry(String txnId, String accountId, EntryType type, 
                       BigDecimal amount, String description) {
        // Immutable record — once created, NEVER modified
    }

    public static void main(String[] args) {
        PaymentLedger ledger = new PaymentLedger();
        
        // Customer pays ₹1000, platform takes 2% fee
        ledger.recordPayment("pay_123", "customer_A", "merchant_B",
            new BigDecimal("1000"), new BigDecimal("20"));
        
        System.out.println("Merchant balance: ₹" + ledger.getBalance("merchant_B")); // ₹980
        System.out.println("Platform revenue: ₹" + ledger.getBalance("platform_revenue")); // ₹20
    }
}
```

---

## Infrastructure Examples

### Stripe's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Language** | Ruby (API), Go (infra), Scala (data) | Core services |
| **Database** | PostgreSQL (sharded), MongoDB | Transaction data, documents |
| **Queue** | Kafka, Redis Streams | Event processing, webhooks |
| **Cache** | Redis, Memcached | Session, idempotency keys |
| **ML** | Python, custom (Radar) | Fraud detection |
| **Infrastructure** | AWS (primary), bare metal | Compute |
| **Encryption** | HSM (Hardware Security Modules) | Key management |
| **Monitoring** | Custom + Datadog | Observability |

### Razorpay's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Language** | Go, PHP (legacy), Python | Core services |
| **Database** | MySQL (sharded), TiDB | Transaction data |
| **Queue** | Kafka, SQS | Async processing |
| **Cache** | Redis | Hot data, rate limiting |
| **ML** | Python + custom | Fraud, routing optimization |
| **Infrastructure** | AWS India | Multi-AZ deployment |
| **Compliance** | PCI DSS Level 1, RBI guidelines | Regulatory |

### Reliability Requirements

```
┌─────────────────────────────────────────────────────────────────┐
│         PAYMENT SYSTEM RELIABILITY REQUIREMENTS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  • Availability: 99.999% (< 5 min downtime per YEAR)            │
│  • Latency: < 2 seconds end-to-end (most < 500ms)             │
│  • Exactly-once semantics (NEVER charge twice)                  │
│  • Data durability: ZERO data loss (every transaction sacred)   │
│  • PCI DSS Level 1 compliance (highest security level)          │
│                                                                   │
│  Stripe processes:                                               │
│  • $1 trillion+ in payment volume per year                      │
│  • Millions of API requests per minute                          │
│  • 99.999% uptime (across all services)                        │
│                                                                   │
│  Razorpay processes:                                             │
│  • $100B+ annual TPV (Total Payment Volume)                     │
│  • 100M+ transactions per month                                 │
│  • Peak: 5000+ transactions per second                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Stripe Handles Black Friday

Black Friday = 10x normal transaction volume for many merchants:

- **Pre-scaling**: Stripe provisions extra capacity weeks before
- **Smart routing**: If Visa network is slow, route through alternative processors
- **Circuit breakers**: If one bank's API is failing, stop sending traffic (don't let it cascade)
- **Graceful degradation**: Non-critical features (analytics dashboard) can lag; payments NEVER lag
- **Multi-region**: US-East down? Failover to US-West in seconds

### Razorpay Handling UPI at Scale (India)

UPI transactions in India present unique challenges:
- **Volume**: 10+ billion UPI transactions per month (India-wide)
- **Latency expectation**: < 5 seconds end-to-end
- **Push model**: UPI uses collect/push flows different from card auth
- **Bank response variability**: Some banks respond in 100ms, others in 30 seconds
- **Timeout handling**: Must handle "pending" state (bank hasn't responded yet)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| No idempotency keys | Network retry = double charge = angry customer + chargeback | ALWAYS use idempotency keys for payment creation |
| Storing credit card numbers | PCI violation, massive security risk, potential $millions in fines | Tokenization: convert to token immediately, never store raw card data |
| Using floating-point for money | 0.1 + 0.2 ≠ 0.3 in IEEE 754 (rounding errors) | Use integers (cents/paise) or BigDecimal. Never float/double for money. |
| Synchronous settlement | Settlement takes days; blocking the user makes no sense | Async settlement pipeline; confirm charge immediately, settle later |
| Single database for ledger | SPOF; any downtime = payments stop | Multi-AZ replicated database with strong consistency |
| Not handling partial failures | Authorization succeeded but capture failed = orphaned hold | State machine with recovery: retry captures, void abandoned auths |

---

## When to Use / When NOT to Use

### When to Build a Custom Payment System
- You ARE a payment company (Stripe, Razorpay, etc.)
- Regulated financial institution requiring full control
- Processing $1B+ annually (cost savings from eliminating middlemen)
- Unique payment flows not supported by existing gateways

### When NOT to Build This
- 99% of companies should USE Stripe/Razorpay, not build their own
- E-commerce startup → Integrate Stripe/Razorpay (days, not months)
- Non-financial company → The compliance cost alone ($500K+ per year for PCI) isn't worth it
- Small volume (< $10M/year) → Processing fees are cheaper than engineering costs

---

## Key Takeaways

1. **Idempotency is non-negotiable**: Every payment API must be idempotent. A network retry should NEVER result in a double charge. This is the most important invariant in payments.
2. **Double-entry ledger** is the backbone: Every money movement has balanced debits and credits, entries are immutable, and you can reconstruct any state at any point in time
3. **The payment state machine** prevents invalid transitions: CREATED → PROCESSING → AUTHORIZED → CAPTURED → SETTLED. No skipping, no going backwards.
4. **Fraud detection in < 100ms**: ML models score transactions instantly using hundreds of signals (device, IP, velocity, amount patterns)
5. **Tokenization for security**: Card numbers never touch the merchant's server. PCI compliance = tokenize at the edge, encrypt at rest, decrypt only in HSMs.
6. **Smart routing** across multiple processors: If Visa is slow, try a different route. If one bank is down, failover to another. This is invisible to the customer.
7. **Never use floating-point for money**: Use integers (smallest currency unit) or BigDecimal. `0.1 + 0.2 ≠ 0.3` in most programming languages.

---

## What's Next?

Next, we'll explore [How Zoom Handles Millions of Video Calls](./09-zoom-video-calls.md) — understanding how real-time video conferencing works at scale with WebRTC, Selective Forwarding Units, and the unique challenges of sub-200ms latency communication.
