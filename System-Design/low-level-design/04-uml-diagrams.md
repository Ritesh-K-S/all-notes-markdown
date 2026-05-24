# UML Diagrams for LLD

UML (Unified Modeling Language) diagrams are the visual tools used to design and communicate class structures, interactions, and workflows in Low Level Design.

---

## 1. Class Diagram

The most important UML diagram for LLD. It shows **classes**, their **attributes**, **methods**, and **relationships**.

### Class Notation

```
┌──────────────────────────────┐
│         ClassName            │  ← Class Name (bold, centered)
├──────────────────────────────┤
│ - privateField: Type         │  ← Attributes (fields)
│ # protectedField: Type       │
│ + publicField: Type          │
├──────────────────────────────┤
│ + publicMethod(): ReturnType │  ← Methods
│ - privateMethod(): void      │
│ # protectedMethod(): Type    │
└──────────────────────────────┘
```

### Access Modifier Symbols

| Symbol | Modifier |
|--------|----------|
| `+` | public |
| `-` | private |
| `#` | protected |
| `~` | package-private (default) |

### Example — Class Diagram for E-Commerce

```
┌──────────────────────────┐
│        <<abstract>>       │
│          Product          │
├──────────────────────────┤
│ - id: String              │
│ - name: String            │
│ - price: double           │
├──────────────────────────┤
│ + getId(): String         │
│ + getName(): String       │
│ + getPrice(): double      │
│ + calculateDiscount():    │
│          double {abstract} │
└────────────┬─────────────┘
             │ extends
    ┌────────┴────────┐
    │                 │
┌───┴──────┐   ┌─────┴──────┐
│Electronics│   │  Clothing  │
├──────────┤   ├────────────┤
│-warranty │   │-size:String│
│ :int     │   │-fabric:    │
├──────────┤   │  String    │
│+calculate│   ├────────────┤
│Discount()│   │+calculate  │
│:double   │   │Discount()  │
└──────────┘   │:double     │
               └────────────┘
```

### Relationship Types in Class Diagrams

```
Association       ─────────────>   (uses / knows about)
Dependency        - - - - - - ->   (temporary usage)
Aggregation       ────────────◇    (has-a, weak — can exist independently)
Composition       ────────────◆    (has-a, strong — cannot exist independently)
Inheritance       ────────────▷    (is-a)
Implementation    - - - - - - ▷    (implements interface)
```

### Multiplicity (Cardinality)

```
1       → Exactly one
0..1    → Zero or one
*       → Zero or more
1..*    → One or more
3..5    → Three to five
```

### Example Relationships

```
┌──────────┐    1      * ┌──────────┐
│Department├──────────────┤Employee  │   Department HAS MANY Employees
│          │  contains    │          │   (Composition — Employee can't exist
└──────────┘              └──────────┘    without Department)

┌──────────┐    *      * ┌──────────┐
│ Student  ├──────────────┤ Course   │   Student ENROLLS in many Courses
│          │  enrolls     │          │   (Association — both exist independently)
└──────────┘              └──────────┘

┌──────────┐    1      * ┌──────────┐
│   Car    ├──────────────┤  Wheel   │   Car HAS 4 Wheels
│          │   has        │          │   (Composition — Wheel belongs to Car)
└──────────┘              └──────────┘
```

### Full Class Diagram Example in Java Mapping

```
┌─────────────────────────────┐
│      <<interface>>          │
│     PaymentMethod           │
├─────────────────────────────┤
│ + pay(amount: double): void │
│ + refund(txnId: String):    │
│            boolean          │
└──────────┬──────────────────┘
           │ implements
    ┌──────┴──────┐
    │             │
┌───┴───────┐ ┌──┴──────────┐
│CreditCard │ │   UPI       │
│Payment    │ │  Payment    │
├───────────┤ ├─────────────┤
│-cardNumber│ │-upiId:String│
│  :String  │ ├─────────────┤
│-cvv:String│ │+pay(): void │
├───────────┤ │+refund():   │
│+pay():void│ │  boolean    │
│+refund(): │ └─────────────┘
│  boolean  │
└───────────┘
```

Corresponding Java Code:

```java
public interface PaymentMethod {
    void pay(double amount);
    boolean refund(String txnId);
}

public class CreditCardPayment implements PaymentMethod {
    private String cardNumber;
    private String cvv;

    @Override
    public void pay(double amount) { /* ... */ }

    @Override
    public boolean refund(String txnId) { return true; }
}

public class UPIPayment implements PaymentMethod {
    private String upiId;

    @Override
    public void pay(double amount) { /* ... */ }

    @Override
    public boolean refund(String txnId) { return true; }
}
```

---

## 2. Sequence Diagram

Shows **how objects interact over time** — the order of method calls between objects for a specific use case.

### Notation

```
┌──────┐          ┌──────┐          ┌──────┐
│ObjectA│         │ObjectB│         │ObjectC│
└──┬───┘          └──┬───┘          └──┬───┘
   │                 │                 │
   │  methodCall()   │                 │       ─── Synchronous call
   │────────────────>│                 │
   │                 │  anotherCall()  │
   │                 │────────────────>│       ─── Nested call
   │                 │                 │
   │                 │  return result  │
   │                 │<────────────────│       ─── Return
   │  return result  │                 │
   │<────────────────│                 │
   │                 │                 │
```

### Example — User Login Flow

```
┌──────┐     ┌──────────┐     ┌──────────┐     ┌────────┐
│Client│     │Controller│     │AuthService│     │UserRepo│
└──┬───┘     └────┬─────┘     └────┬─────┘     └───┬────┘
   │              │                │                │
   │ POST /login  │                │                │
   │─────────────>│                │                │
   │              │ authenticate() │                │
   │              │───────────────>│                │
   │              │                │ findByEmail()  │
   │              │                │───────────────>│
   │              │                │                │
   │              │                │  User object   │
   │              │                │<───────────────│
   │              │                │                │
   │              │                │ verify password│
   │              │                │ (self)         │
   │              │                │                │
   │              │                │ generate JWT   │
   │              │                │ (self)         │
   │              │                │                │
   │              │  JWT Token     │                │
   │              │<───────────────│                │
   │  200 OK      │                │                │
   │  {token}     │                │                │
   │<─────────────│                │                │
```

### Sequence Diagram Elements

| Element | Description |
|---------|-------------|
| **Lifeline** | Vertical dashed line below each object |
| **Activation Bar** | Thin rectangle on lifeline showing active processing |
| **Synchronous Message** | Solid arrow → (waits for response) |
| **Asynchronous Message** | Open arrow → (doesn't wait) |
| **Return Message** | Dashed arrow ← |
| **Self-Call** | Arrow looping back to same lifeline |
| **Alt Fragment** | If-else condition block |
| **Loop Fragment** | Repeated interaction |

### Fragments (Conditions & Loops)

```
   │              │
   │  ┌───────────────────┐
   │  │ alt [valid user]  │
   │  │  return token     │
   │  │<──────────────────│
   │  ├───────────────────┤
   │  │ [invalid user]    │
   │  │  return 401       │
   │  │<──────────────────│
   │  └───────────────────┘
   │              │
```

---

## 3. Use Case Diagram

Shows **what the system does** from the user's perspective. Identifies **actors** and **use cases**.

### Notation

```
     ┌─────────────────────────────────────┐
     │           System Boundary           │
     │                                     │
     │    ┌─────────────────┐              │
  Actor──>│  Use Case 1      │              │
     │    └─────────────────┘              │
     │                                     │
     │    ┌─────────────────┐              │
  Actor──>│  Use Case 2      │              │
     │    └─────────────────┘              │
     │                                     │
     └─────────────────────────────────────┘
```

### Example — Library Management System

```
     ┌───────────────────────────────────────────┐
     │         Library Management System          │
     │                                            │
     │    (Search Book)                           │
     │    (Borrow Book)                           │
  Member──>(Return Book)                          │
     │    (Pay Fine)                              │
     │    (View History)                          │
     │                                            │
     │    (Add Book)                              │
  Librarian>(Remove Book)                         │
     │    (Issue Book)                            │
     │    (Manage Members)                        │
     │                                            │
     │    (Generate Reports)                      │
  Admin──>(Manage Librarians)                     │
     │    (System Configuration)                  │
     │                                            │
     └───────────────────────────────────────────┘
```

### Use Case Relationships

| Relationship | Symbol | Meaning |
|-------------|--------|---------|
| **Include** | `<<include>>` | Use case always includes another (mandatory) |
| **Extend** | `<<extend>>` | Use case optionally extends another |
| **Generalization** | ——▷ | Actor/use case inherits from another |

```
  (Place Order) ──<<include>>──> (Verify Payment)
                                      │
  (Place Order) ──<<extend>>──> (Apply Coupon)
```

- **Include** = Placing an order ALWAYS verifies payment
- **Extend** = Applying a coupon is OPTIONAL during order placement

---

## 4. Activity Diagram (Bonus)

Shows the **flow of activities** — like a flowchart but with support for parallel activities.

### Common Symbols

| Symbol | Meaning |
|--------|---------|
| ● (filled circle) | Start |
| ◉ (circle in circle) | End |
| Rectangle (rounded) | Activity/Action |
| Diamond | Decision (if-else) |
| Thick horizontal bar | Fork/Join (parallel) |

### Example — Order Processing Flow

```
        ●
        │
        ▼
  ┌─────────────┐
  │ Receive Order│
  └──────┬──────┘
         ▼
    ◇ In Stock?
   / \
  Yes  No
  │     │
  ▼     ▼
┌────┐ ┌──────────┐
│Pack│ │Notify     │
│Item│ │Out of     │
└─┬──┘ │Stock      │
  │    └─────┬────┘
  │          │
  ▼          ▼
┌──────┐    ◉
│Ship  │
└──┬───┘
   ▼
┌────────────┐
│Send Tracking│
│Email        │
└──────┬─────┘
       ▼
       ◉
```

---

## UML in LLD Interviews — Tips

1. **Start with Use Case Diagram** — Identify actors and features
2. **Draw Class Diagram** — Define entities, attributes, methods, relationships
3. **Draw Sequence Diagram** — For 1-2 key flows (e.g., happy path)
4. **Keep it clean** — Don't clutter with every field; show key ones
5. **Show relationships clearly** — Composition vs Aggregation vs Association
6. **Label multiplicities** — 1:1, 1:*, *:*
7. **Mark abstract classes and interfaces** — Use `<<abstract>>` and `<<interface>>`

---

## Tools for Drawing UML

| Tool | Type |
|------|------|
| **draw.io / diagrams.net** | Free, web-based |
| **PlantUML** | Text-to-diagram |
| **Lucidchart** | Cloud-based |
| **Mermaid** | Markdown-compatible |
| **StarUML** | Desktop app |
| **Pen & Paper** | Interview-friendly |
