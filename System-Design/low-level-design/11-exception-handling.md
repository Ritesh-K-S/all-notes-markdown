# Exception Handling in LLD

Proper exception handling is a key part of designing robust, maintainable systems. It defines how errors are communicated, propagated, and handled across layers.

---

## 1. Exception Hierarchy in Java

```
Throwable
├── Error (JVM-level — don't catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
│
└── Exception
    ├── Checked Exceptions (compile-time — must handle)
    │   ├── IOException
    │   ├── SQLException
    │   └── ClassNotFoundException
    │
    └── RuntimeException (unchecked — optional to handle)
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── IllegalStateException
        ├── IndexOutOfBoundsException
        └── UnsupportedOperationException
```

### Checked vs Unchecked

| Feature | Checked | Unchecked (Runtime) |
|---------|---------|-------------------|
| Enforced by compiler | Yes | No |
| Must catch or declare | Yes | No |
| Use for | Recoverable errors | Programming bugs |
| Example | FileNotFoundException | NullPointerException |

---

## 2. Custom Exception Hierarchy

In LLD, design a **custom exception hierarchy** specific to your domain.

### Base Exception

```java
// Base exception for the entire application
public class ApplicationException extends RuntimeException {
    private final String errorCode;

    public ApplicationException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public ApplicationException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}
```

### Domain-Specific Exceptions

```java
// Entity not found
public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resourceName, String id) {
        super(resourceName + " not found with id: " + id, "NOT_FOUND");
    }
}

// Business rule violation
public class BusinessRuleException extends ApplicationException {
    public BusinessRuleException(String message) {
        super(message, "BUSINESS_RULE_VIOLATION");
    }
}

// Validation error
public class ValidationException extends ApplicationException {
    private final List<FieldError> errors;

    public ValidationException(List<FieldError> errors) {
        super("Validation failed", "VALIDATION_ERROR");
        this.errors = errors;
    }

    public List<FieldError> getErrors() {
        return errors;
    }
}

public class FieldError {
    private final String field;
    private final String message;

    public FieldError(String field, String message) {
        this.field = field;
        this.message = message;
    }

    public String getField() { return field; }
    public String getMessage() { return message; }
}

// Authentication / Authorization
public class UnauthorizedException extends ApplicationException {
    public UnauthorizedException(String message) {
        super(message, "UNAUTHORIZED");
    }
}

public class ForbiddenException extends ApplicationException {
    public ForbiddenException(String message) {
        super(message, "FORBIDDEN");
    }
}

// Duplicate / Conflict
public class DuplicateResourceException extends ApplicationException {
    public DuplicateResourceException(String resourceName, String field, String value) {
        super(resourceName + " already exists with " + field + ": " + value, "CONFLICT");
    }
}
```

### Full Exception Hierarchy

```
ApplicationException
├── ResourceNotFoundException
├── BusinessRuleException
├── ValidationException
├── UnauthorizedException
├── ForbiddenException
├── DuplicateResourceException
└── ExternalServiceException
```

---

## 3. Exception Handling Best Practices

### DO: Throw Early, Catch Late

```java
// Throw early — at the point of detection
public class UserService {
    public User getUser(String userId) {
        if (userId == null || userId.isBlank()) {
            throw new IllegalArgumentException("User ID cannot be null or empty");
        }
        return userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", userId));
    }
}

// Catch late — at the boundary (controller/handler)
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable String id) {
        // Don't catch here — let global handler deal with it
        User user = userService.getUser(id);
        return ResponseEntity.ok(UserMapper.toResponse(user));
    }
}
```

### DO: Use Specific Exceptions

```java
// Bad — too generic
throw new Exception("Something went wrong");
throw new RuntimeException("Error");

// Good — specific and informative
throw new ResourceNotFoundException("Order", orderId);
throw new BusinessRuleException("Cannot cancel a delivered order");
throw new ValidationException(List.of(
    new FieldError("email", "Invalid email format")
));
```

### DON'T: Catch and Swallow

```java
// TERRIBLE — hides errors completely
try {
    processPayment(order);
} catch (Exception e) {
    // Do nothing — silent failure!
}

// BAD — only logs, doesn't propagate
try {
    processPayment(order);
} catch (Exception e) {
    log.error("Error", e);
    // Caller has no idea it failed!
}

// GOOD — log and re-throw or throw meaningful exception
try {
    processPayment(order);
} catch (PaymentGatewayException e) {
    log.error("Payment failed for order: {}", order.getId(), e);
    throw new BusinessRuleException("Payment processing failed. Please try again.");
}
```

### DON'T: Use Exceptions for Flow Control

```java
// BAD — using exception as if/else
try {
    User user = userRepository.findById(id);
} catch (UserNotFoundException e) {
    user = createDefaultUser();
}

// GOOD — use Optional or conditional check
Optional<User> userOpt = userRepository.findById(id);
User user = userOpt.orElseGet(() -> createDefaultUser());
```

### DON'T: Catch Generic Exception

```java
// BAD
try {
    // Code that can throw multiple exceptions
} catch (Exception e) {
    // What kind of error? No idea.
}

// GOOD — catch specific exceptions
try {
    // Code
} catch (ResourceNotFoundException e) {
    // Handle not found
} catch (ValidationException e) {
    // Handle validation errors
} catch (Exception e) {
    // Last resort — unexpected errors only
    log.error("Unexpected error", e);
    throw e;
}
```

---

## 4. Global Exception Handler

Centralized error handling at the application boundary.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            ex.getErrors(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<ErrorResponse> handleBusinessRule(BusinessRuleException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(error);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(UnauthorizedException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse("UNAUTHORIZED", ex.getMessage(), LocalDateTime.now()));
    }

    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);
        ErrorResponse error = new ErrorResponse(
            "INTERNAL_ERROR",
            "An unexpected error occurred",  // Don't expose internal details
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

### Error Response DTO

```java
public class ErrorResponse {
    private String errorCode;
    private String message;
    private List<FieldError> details;
    private LocalDateTime timestamp;

    public ErrorResponse(String errorCode, String message, LocalDateTime timestamp) {
        this.errorCode = errorCode;
        this.message = message;
        this.timestamp = timestamp;
    }

    public ErrorResponse(String errorCode, String message, 
                         List<FieldError> details, LocalDateTime timestamp) {
        this(errorCode, message, timestamp);
        this.details = details;
    }

    // Getters
}
```

---

## 5. Exception Handling Across Layers

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Controller  │     │   Service    │     │  Repository  │     │   External   │
│   (Catch)    │ ←── │  (Throw)     │ ←── │  (Throw)     │ ←── │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       ↓                    ↓                    ↓                     ↓
  Maps to HTTP         Throws domain        Throws data           Throws 3rd
  status codes         exceptions           access errors         party errors
```

```java
// Repository layer — throws data access exceptions
public class UserRepository {
    public User findById(String id) {
        // If not found, return Optional.empty() — don't throw here
        return Optional.ofNullable(database.get(id));
    }
}

// Service layer — throws domain exceptions
public class UserService {
    private final UserRepository userRepository;

    public User getUser(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    public User createUser(CreateUserRequest request) {
        // Validate
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException("User", "email", request.getEmail());
        }
        // Create
        return userRepository.save(UserMapper.toEntity(request));
    }
}

// Controller layer — handled by GlobalExceptionHandler
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable String id) {
        User user = userService.getUser(id);  // Exceptions propagate up
        return ResponseEntity.ok(UserMapper.toResponse(user));
    }
}
```

---

## 6. Try-With-Resources

For objects that implement `AutoCloseable` — ensures resources are closed.

```java
// Old way — verbose
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("data.txt"));
    String line = reader.readLine();
} catch (IOException e) {
    log.error("Error reading file", e);
} finally {
    if (reader != null) {
        try { reader.close(); } catch (IOException e) { /* ignore */ }
    }
}

// Modern way — try-with-resources
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    String line = reader.readLine();
} catch (IOException e) {
    log.error("Error reading file", e);
}
// reader is automatically closed — even if an exception occurs
```

---

## 7. Exception Handling Checklist

- [ ] Custom exception hierarchy defined for the domain
- [ ] Specific exceptions used (not generic Exception)
- [ ] Exceptions are informative (include context — IDs, field names)
- [ ] Global exception handler at the boundary
- [ ] Consistent error response format
- [ ] Internal details not exposed to clients (500 errors)
- [ ] Resources properly closed (try-with-resources)
- [ ] Exceptions not used for flow control
- [ ] Checked exceptions wrapped into domain exceptions at boundaries
- [ ] Exceptions logged at the appropriate level
