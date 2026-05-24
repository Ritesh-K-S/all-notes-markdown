# Testing & TDD in LLD

Writing testable code is a core LLD skill. Tests validate that your design works correctly and remains stable through changes.

---

## 1. Testing Pyramid

```
        ╱  E2E Tests  ╲        ← Few, slow, expensive
       ╱────────────────╲
      ╱ Integration Tests ╲    ← Some, test component interactions
     ╱──────────────────────╲
    ╱     Unit Tests          ╲  ← Many, fast, cheap
   ╱────────────────────────────╲
```

| Level | What It Tests | Speed | Count |
|-------|--------------|-------|-------|
| **Unit** | Single class/method in isolation | Fast (ms) | Many |
| **Integration** | Multiple components together (DB, API) | Medium (s) | Some |
| **E2E** | Full user flow through the system | Slow (min) | Few |

---

## 2. Unit Testing Basics (JUnit 5)

### Structure: AAA Pattern

```java
@Test
void shouldCalculateTotalWithDiscount() {
    // Arrange — setup test data
    Order order = new Order();
    order.addItem(new OrderItem("Laptop", new BigDecimal("1000"), 1));
    order.addItem(new OrderItem("Mouse", new BigDecimal("50"), 2));
    DiscountService discount = new PercentageDiscount(10);

    // Act — execute the behavior
    BigDecimal total = order.calculateTotal(discount);

    // Assert — verify the result
    assertEquals(new BigDecimal("990.00"), total);
}
```

### Common Assertions

```java
assertEquals(expected, actual);
assertNotNull(object);
assertTrue(condition);
assertThrows(NotFoundException.class, () -> service.findById("invalid"));
assertAll(
    () -> assertEquals("John", user.getName()),
    () -> assertEquals("john@mail.com", user.getEmail())
);
```

### Test Naming Conventions

```java
// Pattern: should_ExpectedBehavior_When_Condition
@Test void shouldReturnEmpty_WhenNoItemsInCart() { }

@Test void shouldThrowException_WhenBalanceInsufficient() { }

@Test void shouldApplyDiscount_WhenCouponIsValid() { }
```

---

## 3. Test-Driven Development (TDD)

### Red → Green → Refactor Cycle

```
1. RED    — Write a failing test for the next feature
2. GREEN  — Write minimal code to make the test pass
3. REFACTOR — Clean up code while keeping tests green
```

### TDD Example: Shopping Cart

```java
// Step 1: RED — Write failing test
@Test
void shouldReturnZero_WhenCartIsEmpty() {
    ShoppingCart cart = new ShoppingCart();
    assertEquals(BigDecimal.ZERO, cart.getTotal());
}

// Step 2: GREEN — Minimal implementation
public class ShoppingCart {
    public BigDecimal getTotal() {
        return BigDecimal.ZERO;
    }
}

// Step 3: Next test (RED)
@Test
void shouldReturnItemPrice_WhenOneItemAdded() {
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Book", new BigDecimal("29.99")));
    assertEquals(new BigDecimal("29.99"), cart.getTotal());
}

// Step 4: GREEN — Extend implementation
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) { items.add(item); }

    public BigDecimal getTotal() {
        return items.stream()
            .map(Item::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

---

## 4. Mocking with Mockito

### Why Mock?
- Isolate the class under test from its dependencies
- Simulate different scenarios (errors, edge cases)
- Avoid hitting real databases/APIs in unit tests

### Basic Mocking

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private OrderRepository orderRepository;
    @Mock private PaymentGateway paymentGateway;
    @InjectMocks private OrderService orderService;

    @Test
    void shouldPlaceOrder_WhenPaymentSucceeds() {
        // Arrange
        Order order = new Order("user1", List.of(new Item("Book", 29.99)));
        when(paymentGateway.charge(anyString(), any(BigDecimal.class)))
            .thenReturn(PaymentResult.SUCCESS);
        when(orderRepository.save(any(Order.class)))
            .thenAnswer(inv -> inv.getArgument(0));

        // Act
        Order result = orderService.placeOrder(order);

        // Assert
        assertEquals(OrderStatus.CONFIRMED, result.getStatus());
        verify(paymentGateway).charge(eq("user1"), any(BigDecimal.class));
        verify(orderRepository).save(order);
    }

    @Test
    void shouldMarkFailed_WhenPaymentFails() {
        Order order = new Order("user1", List.of(new Item("Book", 29.99)));
        when(paymentGateway.charge(anyString(), any()))
            .thenReturn(PaymentResult.DECLINED);

        Order result = orderService.placeOrder(order);

        assertEquals(OrderStatus.PAYMENT_FAILED, result.getStatus());
        verify(orderRepository, never()).save(any());
    }
}
```

### Verify Interactions

```java
verify(mock).method(args);              // Called exactly once
verify(mock, times(2)).method(args);    // Called exactly twice
verify(mock, never()).method(args);     // Never called
verify(mock, atLeast(1)).method(args);  // Called at least once
verifyNoMoreInteractions(mock);         // No other methods called
```

---

## 5. Writing Testable Code (Design for Testability)

### Bad: Hard to Test

```java
public class OrderService {
    // ❌ Direct instantiation — can't mock
    private PaymentGateway gateway = new StripePaymentGateway();

    // ❌ Static method call — can't mock
    public void process(Order order) {
        if (DateUtils.isWeekend()) {  // static dependency
            throw new UnsupportedOperationException();
        }
        gateway.charge(order);
    }
}
```

### Good: Testable via Dependency Injection

```java
public class OrderService {
    private final PaymentGateway gateway;        // ✅ Interface
    private final Clock clock;                    // ✅ Injectable clock

    public OrderService(PaymentGateway gateway, Clock clock) {
        this.gateway = gateway;                   // ✅ Injected
        this.clock = clock;
    }

    public void process(Order order) {
        if (isWeekend(clock)) {
            throw new UnsupportedOperationException();
        }
        gateway.charge(order);
    }
}
```

### Key Principles

| Principle | Why |
|-----------|-----|
| **Inject dependencies** | Allows mocking in tests |
| **Program to interfaces** | Can swap real impl with test doubles |
| **Avoid static methods** | Can't be mocked easily |
| **Small methods** | Easier to test individual behaviors |
| **Single Responsibility** | One reason to change = one set of tests |

---

## 6. Test Doubles

| Type | Purpose | Example |
|------|---------|---------|
| **Dummy** | Fill parameter, never used | `new DummyLogger()` |
| **Stub** | Returns canned answers | `when(repo.findById(1)).thenReturn(user)` |
| **Mock** | Verifies interactions | `verify(emailService).send(any())` |
| **Spy** | Real object with some methods overridden | `spy(realService)` |
| **Fake** | Working lightweight implementation | In-memory repository |

### Fake Example

```java
// Fake repository for testing (no database needed)
public class InMemoryUserRepository implements UserRepository {
    private final Map<String, User> store = new HashMap<>();

    public User save(User user) {
        store.put(user.getId(), user);
        return user;
    }

    public Optional<User> findById(String id) {
        return Optional.ofNullable(store.get(id));
    }

    public List<User> findAll() {
        return new ArrayList<>(store.values());
    }
}
```

---

## 7. Testing Patterns for LLD

### Testing State Machines

```java
@Test
void orderStateMachine_ShouldFollowValidTransitions() {
    Order order = new Order();
    assertEquals(OrderStatus.CREATED, order.getStatus());

    order.confirm();
    assertEquals(OrderStatus.CONFIRMED, order.getStatus());

    order.ship();
    assertEquals(OrderStatus.SHIPPED, order.getStatus());

    order.deliver();
    assertEquals(OrderStatus.DELIVERED, order.getStatus());
}

@Test
void shouldNotAllowInvalidTransition() {
    Order order = new Order();  // Status = CREATED
    assertThrows(InvalidStateTransitionException.class, () -> order.ship());
}
```

### Testing Strategy Pattern

```java
@ParameterizedTest
@MethodSource("discountStrategies")
void shouldApplyCorrectDiscount(DiscountStrategy strategy,
                                 BigDecimal price, BigDecimal expected) {
    assertEquals(expected, strategy.apply(price));
}

static Stream<Arguments> discountStrategies() {
    return Stream.of(
        Arguments.of(new FlatDiscount(10), bd("100"), bd("90")),
        Arguments.of(new PercentageDiscount(20), bd("100"), bd("80")),
        Arguments.of(new NoDiscount(), bd("100"), bd("100"))
    );
}
```

---

## Testing Checklist

- [ ] Each public method has at least one test
- [ ] Happy path AND edge cases are covered
- [ ] Exception scenarios are tested with `assertThrows`
- [ ] Dependencies are mocked (not real DB/API in unit tests)
- [ ] Tests are independent — no shared mutable state
- [ ] Test names clearly describe the scenario
- [ ] No logic in tests (no if/loops)
- [ ] Tests run fast (unit tests < 1 second total)
