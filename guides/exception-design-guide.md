# Effective Exception Design and Handling in Java

A comprehensive guide to designing robust exception hierarchies and handling errors effectively in enterprise Java applications.

---

## Table of Contents

1. [Exception Fundamentals](#exception-fundamentals)
2. [Exception Hierarchy Design](#exception-hierarchy-design)
3. [Checked vs Unchecked Exceptions](#checked-vs-unchecked-exceptions)
4. [Exception Handling Patterns](#exception-handling-patterns)
5. [Error Messages and Context](#error-messages-and-context)
6. [Exception Translation](#exception-translation)
7. [Try-with-Resources](#try-with-resources)
8. [Async Exception Handling](#async-exception-handling)
9. [Logging and Monitoring](#logging-and-monitoring)
10. [Testing Exceptions](#testing-exceptions)
11. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
12. [Framework-Specific Handling](#framework-specific-handling)
13. [Decision Tree](#decision-tree)

---

## Exception Fundamentals

### The Exception Hierarchy

```java
/**
 * Java Exception Hierarchy:
 * 
 * Throwable
 * ├── Error (unchecked)
 * │   ├── OutOfMemoryError
 * │   ├── StackOverflowError
 * │   └── VirtualMachineError
 * │
 * └── Exception
 *     ├── RuntimeException (unchecked)
 *     │   ├── NullPointerException
 *     │   ├── IllegalArgumentException
 *     │   ├── IllegalStateException
 *     │   └── IndexOutOfBoundsException
 *     │
 *     └── Checked Exceptions
 *         ├── IOException
 *         ├── SQLException
 *         └── ClassNotFoundException
 */

public class ExceptionBasics {
    
    /**
     * Key Principles:
     * 
     * 1. Errors - Fatal, unrecoverable (e.g., OutOfMemoryError)
     *    → Don't catch, let application crash
     * 
     * 2. Checked Exceptions - Recoverable conditions (e.g., FileNotFoundException)
     *    → Must be handled or declared
     * 
     * 3. Unchecked (Runtime) Exceptions - Programming errors (e.g., NullPointerException)
     *    → Don't need to be declared
     */
}
```

### When to Use Exceptions

```java
public class WhenToUseExceptions {
    
    // ✅ GOOD - Use exceptions for exceptional conditions
    public User getUser(String id) {
        User user = database.findById(id);
        if (user == null) {
            throw new UserNotFoundException("User not found: " + id);
        }
        return user;
    }
    
    // ❌ BAD - Don't use exceptions for flow control
    public boolean isValidEmail(String email) {
        try {
            new InternetAddress(email).validate();
            return true;
        } catch (AddressException e) {
            return false;  // Expensive for flow control!
        }
    }
    
    // ✅ BETTER - Use return value for expected conditions
    public boolean isValidEmailBetter(String email) {
        return email != null && email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
    
    /**
     * Use exceptions when:
     * ✅ Condition is truly exceptional (rare)
     * ✅ Caller can't reasonably check beforehand
     * ✅ Error needs to propagate up the call stack
     * ✅ Need rich error context
     * 
     * Don't use exceptions when:
     * ❌ Condition is expected/common (use Optional, Result type, or boolean)
     * ❌ For flow control (if/else is clearer and faster)
     * ❌ In performance-critical code (exceptions are expensive)
     */
}
```

---

## Exception Hierarchy Design

### 1. **Domain-Specific Exception Hierarchy**

```java
/**
 * Well-designed exception hierarchy for an e-commerce system
 */

// Base exception for entire application
public abstract class EcommerceException extends RuntimeException {
    
    private final ErrorCode errorCode;
    private final Map<String, Object> context;
    
    protected EcommerceException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }
    
    protected EcommerceException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }
    
    public EcommerceException addContext(String key, Object value) {
        this.context.put(key, value);
        return this;
    }
    
    public ErrorCode getErrorCode() {
        return errorCode;
    }
    
    public Map<String, Object> getContext() {
        return Collections.unmodifiableMap(context);
    }
    
    // HTTP status for REST APIs
    public abstract int getHttpStatus();
    
    // Whether to retry operation
    public boolean isRetryable() {
        return false;
    }
}

// Domain-specific exception categories
public abstract class ValidationException extends EcommerceException {
    protected ValidationException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }
    
    @Override
    public int getHttpStatus() {
        return 400;  // Bad Request
    }
}

public abstract class ResourceException extends EcommerceException {
    protected ResourceException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }
    
    protected ResourceException(ErrorCode errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
    
    @Override
    public int getHttpStatus() {
        return 404;  // Not Found
    }
}

public abstract class BusinessException extends EcommerceException {
    protected BusinessException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }
    
    @Override
    public int getHttpStatus() {
        return 422;  // Unprocessable Entity
    }
}

public abstract class InfrastructureException extends EcommerceException {
    protected InfrastructureException(ErrorCode errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
    
    @Override
    public int getHttpStatus() {
        return 503;  // Service Unavailable
    }
    
    @Override
    public boolean isRetryable() {
        return true;  // Infrastructure issues are often transient
    }
}

// Concrete exceptions
public class InvalidEmailException extends ValidationException {
    public InvalidEmailException(String email) {
        super(ErrorCode.INVALID_EMAIL, "Invalid email format: " + email);
        addContext("email", email);
    }
}

public class UserNotFoundException extends ResourceException {
    public UserNotFoundException(String userId) {
        super(ErrorCode.USER_NOT_FOUND, "User not found with ID: " + userId);
        addContext("userId", userId);
    }
}

public class InsufficientInventoryException extends BusinessException {
    public InsufficientInventoryException(String productId, int requested, int available) {
        super(ErrorCode.INSUFFICIENT_INVENTORY, 
            String.format("Insufficient inventory for product %s. Requested: %d, Available: %d",
                productId, requested, available));
        addContext("productId", productId);
        addContext("requested", requested);
        addContext("available", available);
    }
}

public class PaymentGatewayException extends InfrastructureException {
    public PaymentGatewayException(String message, Throwable cause) {
        super(ErrorCode.PAYMENT_GATEWAY_ERROR, message, cause);
    }
}

// Error codes enum
public enum ErrorCode {
    // Validation errors (4xx)
    INVALID_EMAIL("E1001", "Invalid email address"),
    INVALID_PHONE("E1002", "Invalid phone number"),
    INVALID_INPUT("E1003", "Invalid input"),
    
    // Resource errors (4xx)
    USER_NOT_FOUND("E2001", "User not found"),
    ORDER_NOT_FOUND("E2002", "Order not found"),
    PRODUCT_NOT_FOUND("E2003", "Product not found"),
    
    // Business errors (4xx)
    INSUFFICIENT_INVENTORY("E3001", "Insufficient inventory"),
    PAYMENT_DECLINED("E3002", "Payment declined"),
    ORDER_ALREADY_SHIPPED("E3003", "Order already shipped"),
    
    // Infrastructure errors (5xx)
    DATABASE_ERROR("E4001", "Database error"),
    PAYMENT_GATEWAY_ERROR("E4002", "Payment gateway error"),
    EXTERNAL_SERVICE_ERROR("E4003", "External service error");
    
    private final String code;
    private final String description;
    
    ErrorCode(String code, String description) {
        this.code = code;
        this.description = description;
    }
    
    public String getCode() { return code; }
    public String getDescription() { return description; }
}
```

### 2. **Builder Pattern for Complex Exceptions**

```java
public class OrderProcessingException extends BusinessException {
    
    private final String orderId;
    private final String customerId;
    private final OrderStatus currentStatus;
    private final List<String> validationErrors;
    
    private OrderProcessingException(Builder builder) {
        super(builder.errorCode, builder.message);
        this.orderId = builder.orderId;
        this.customerId = builder.customerId;
        this.currentStatus = builder.currentStatus;
        this.validationErrors = List.copyOf(builder.validationErrors);
        
        // Add all to context
        addContext("orderId", orderId);
        addContext("customerId", customerId);
        addContext("currentStatus", currentStatus);
        addContext("validationErrors", validationErrors);
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private ErrorCode errorCode;
        private String message;
        private String orderId;
        private String customerId;
        private OrderStatus currentStatus;
        private List<String> validationErrors = new ArrayList<>();
        
        public Builder errorCode(ErrorCode errorCode) {
            this.errorCode = errorCode;
            return this;
        }
        
        public Builder message(String message) {
            this.message = message;
            return this;
        }
        
        public Builder orderId(String orderId) {
            this.orderId = orderId;
            return this;
        }
        
        public Builder customerId(String customerId) {
            this.customerId = customerId;
            return this;
        }
        
        public Builder currentStatus(OrderStatus currentStatus) {
            this.currentStatus = currentStatus;
            return this;
        }
        
        public Builder addValidationError(String error) {
            this.validationErrors.add(error);
            return this;
        }
        
        public OrderProcessingException build() {
            Objects.requireNonNull(errorCode, "errorCode is required");
            Objects.requireNonNull(message, "message is required");
            return new OrderProcessingException(this);
        }
    }
    
    public String getOrderId() { return orderId; }
    public String getCustomerId() { return customerId; }
    public OrderStatus getCurrentStatus() { return currentStatus; }
    public List<String> getValidationErrors() { return validationErrors; }
}

// Usage
throw OrderProcessingException.builder()
    .errorCode(ErrorCode.INVALID_INPUT)
    .message("Order validation failed")
    .orderId("ORD-123")
    .customerId("CUST-456")
    .currentStatus(OrderStatus.PENDING)
    .addValidationError("Invalid shipping address")
    .addValidationError("Invalid payment method")
    .build();
```

---

## Checked vs Unchecked Exceptions

### Decision Framework

```java
public class CheckedVsUnchecked {
    
    /**
     * Use CHECKED exceptions when:
     * ✅ Caller can reasonably recover from the error
     * ✅ Condition is external and not under caller's control
     * ✅ You want to force callers to handle it
     * 
     * Examples:
     * - FileNotFoundException (file might not exist, caller should handle)
     * - IOException (network might be down, retry possible)
     * - SQLException (database might be unavailable, fallback possible)
     */
    
    // Example: Checked exception for external resource
    public class ExternalServiceException extends Exception {
        public ExternalServiceException(String message, Throwable cause) {
            super(message, cause);
        }
    }
    
    public User fetchUserFromExternalService(String id) throws ExternalServiceException {
        try {
            return externalApi.getUser(id);
        } catch (IOException e) {
            throw new ExternalServiceException("Failed to fetch user from external service", e);
        }
    }
    
    /**
     * Use UNCHECKED (Runtime) exceptions when:
     * ✅ Error indicates a programming bug (should be fixed, not handled)
     * ✅ Caller cannot reasonably recover
     * ✅ Error is due to invalid arguments or state
     * 
     * Examples:
     * - NullPointerException (programming error)
     * - IllegalArgumentException (invalid input, should be validated)
     * - IllegalStateException (object in wrong state, design issue)
     */
    
    // Example: Unchecked exception for programming errors
    public class InvalidOrderStateException extends RuntimeException {
        public InvalidOrderStateException(String message) {
            super(message);
        }
    }
    
    public void shipOrder(Order order) {
        if (order.getStatus() != OrderStatus.PAID) {
            throw new InvalidOrderStateException(
                "Cannot ship order in status: " + order.getStatus()
            );
        }
        // Ship order
    }
}
```

### Modern Trend: Prefer Unchecked

```java
public class ModernExceptionDesign {
    
    /**
     * Modern best practice: Use unchecked exceptions
     * 
     * Reasons:
     * 1. Checked exceptions pollute method signatures
     * 2. Force unnecessary try-catch blocks
     * 3. Break lambda expressions and streams
     * 4. Make refactoring harder
     * 
     * Most modern frameworks (Spring, Hibernate) use unchecked exceptions
     */
    
    // ❌ OLD STYLE - Checked exceptions
    public User getUserOldStyle(String id) throws UserNotFoundException, DatabaseException {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // Forces ugly handling:
    public void oldStyleUsage() {
        try {
            User user = getUserOldStyle("123");
        } catch (UserNotFoundException | DatabaseException e) {
            // Forced to handle even if can't do anything meaningful
        }
    }
    
    // ✅ MODERN STYLE - Unchecked exceptions
    public User getUser(String id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    // Clean usage:
    public void modernStyleUsage() {
        User user = getUser("123");  // Let exception bubble up
        // Or handle at appropriate level
    }
    
    // Works beautifully with streams:
    public List<User> getUsers(List<String> ids) {
        return ids.stream()
            .map(this::getUser)  // Can't do this with checked exceptions!
            .collect(Collectors.toList());
    }
}
```

---

## Exception Handling Patterns

### 1. **Handle at Appropriate Level**

```java
public class ExceptionHandlingLevels {
    
    // ❌ BAD - Catch too early, lose context
    public class UserService {
        public User getUser(String id) {
            try {
                return userRepository.findById(id);
            } catch (Exception e) {
                logger.error("Error getting user", e);
                return null;  // Swallow exception, return null
            }
        }
    }
    
    // ✅ GOOD - Let exception propagate, handle at boundary
    public class UserService {
        public User getUser(String id) {
            return userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
            // Let exception bubble up
        }
    }
    
    // Handle at REST controller (boundary)
    @RestController
    public class UserController {
        
        private final UserService userService;
        
        @GetMapping("/users/{id}")
        public ResponseEntity<User> getUser(@PathVariable String id) {
            try {
                User user = userService.getUser(id);
                return ResponseEntity.ok(user);
            } catch (UserNotFoundException e) {
                return ResponseEntity.notFound().build();
            }
        }
    }
    
    /**
     * Exception handling levels:
     * 
     * 1. Don't catch - Let propagate if you can't handle meaningfully
     * 2. Translate - Convert low-level to high-level exception
     * 3. Add context - Wrap with more information
     * 4. Handle - Only at boundaries (Controller, API Gateway, Main)
     */
}
```

### 2. **Exception Translation**

```java
public class ExceptionTranslation {
    
    // Translate infrastructure exceptions to domain exceptions
    @Repository
    public class UserRepository {
        
        public User findById(String id) {
            try {
                return jdbcTemplate.queryForObject(
                    "SELECT * FROM users WHERE id = ?",
                    new Object[]{id},
                    userRowMapper
                );
            } catch (EmptyResultDataAccessException e) {
                // Translate Spring exception to domain exception
                throw new UserNotFoundException(id);
            } catch (DataAccessException e) {
                // Translate to infrastructure exception
                throw new DatabaseException("Failed to fetch user", e)
                    .addContext("userId", id)
                    .addContext("operation", "findById");
            }
        }
    }
    
    // Service layer doesn't know about database exceptions
    @Service
    public class UserService {
        
        private final UserRepository userRepository;
        
        public User getUser(String id) {
            // Works with domain exceptions only
            return userRepository.findById(id);
        }
    }
}
```

### 3. **Try-Catch-Finally Best Practices**

```java
public class TryCatchFinally {
    
    // ❌ BAD - Empty catch block
    public void badExample() {
        try {
            riskyOperation();
        } catch (Exception e) {
            // Silent failure - never do this!
        }
    }
    
    // ✅ GOOD - At minimum, log the exception
    public void goodExample() {
        try {
            riskyOperation();
        } catch (Exception e) {
            logger.error("Failed to perform risky operation", e);
            throw e;  // Re-throw if can't handle
        }
    }
    
    // ✅ GOOD - Resource cleanup in finally
    public String readFile(String path) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader(path));
            return reader.readLine();
        } catch (IOException e) {
            throw new FileReadException("Failed to read file: " + path, e);
        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (IOException e) {
                    logger.warn("Failed to close reader", e);
                }
            }
        }
    }
    
    // ✅ BETTER - Use try-with-resources (covered later)
    public String readFileBetter(String path) {
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            return reader.readLine();
        } catch (IOException e) {
            throw new FileReadException("Failed to read file: " + path, e);
        }
    }
    
    // Multiple catch blocks - order matters!
    public void multipleCatch() {
        try {
            riskyOperation();
        } catch (FileNotFoundException e) {
            // Most specific first
            logger.error("File not found", e);
        } catch (IOException e) {
            // More general
            logger.error("I/O error", e);
        } catch (Exception e) {
            // Most general last
            logger.error("Unexpected error", e);
        }
    }
    
    // Java 7+ - Multi-catch
    public void multiCatch() {
        try {
            riskyOperation();
        } catch (IOException | SQLException e) {
            logger.error("Data access error", e);
            throw new DataAccessException("Data access failed", e);
        }
    }
}
```

### 4. **Suppressed Exceptions**

```java
public class SuppressedExceptions {
    
    // Problem: Exception in finally can hide original exception
    public void problematicFinally() {
        try {
            throw new RuntimeException("Original exception");
        } finally {
            throw new RuntimeException("Finally exception");  // Hides original!
        }
    }
    
    // Solution 1: Try-with-resources (automatic suppression)
    public void trySolution() {
        try (MyResource resource = new MyResource()) {
            throw new RuntimeException("Main exception");
        }
        // If close() throws, it's suppressed and attached to main exception
    }
    
    static class MyResource implements AutoCloseable {
        @Override
        public void close() {
            throw new RuntimeException("Close exception");
        }
    }
    
    // Solution 2: Manual suppression
    public void manualSuppression() {
        Exception mainException = null;
        try {
            throw new RuntimeException("Main exception");
        } catch (Exception e) {
            mainException = e;
            throw e;
        } finally {
            try {
                cleanup();
            } catch (Exception e) {
                if (mainException != null) {
                    mainException.addSuppressed(e);
                } else {
                    throw e;
                }
            }
        }
    }
    
    // Retrieve suppressed exceptions
    public void getSuppressed() {
        try {
            // Code that might suppress exceptions
        } catch (Exception e) {
            logger.error("Main exception", e);
            for (Throwable suppressed : e.getSuppressed()) {
                logger.warn("Suppressed exception", suppressed);
            }
        }
    }
}
```

---

## Error Messages and Context

### 1. **Informative Error Messages**

```java
public class ErrorMessages {
    
    // ❌ BAD - Vague, no context
    public void badMessages() {
        throw new RuntimeException("Error");
        throw new IllegalArgumentException("Invalid input");
        throw new IllegalStateException("Wrong state");
    }
    
    // ✅ GOOD - Specific, actionable, with context
    public void goodMessages(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Order cannot be null");
        }
        
        if (order.getItems().isEmpty()) {
            throw new IllegalArgumentException(
                "Order must contain at least one item. Order ID: " + order.getId()
            );
        }
        
        if (order.getStatus() != OrderStatus.PENDING) {
            throw new IllegalStateException(
                String.format(
                    "Cannot modify order in status %s. Order must be in PENDING status. Order ID: %s",
                    order.getStatus(),
                    order.getId()
                )
            );
        }
    }
    
    // ✅ BEST - Include all relevant context
    public void processPayment(Payment payment) {
        if (payment.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidPaymentException(
                String.format(
                    "Payment amount must be positive. " +
                    "Payment ID: %s, Amount: %s, Currency: %s, Customer: %s",
                    payment.getId(),
                    payment.getAmount(),
                    payment.getCurrency(),
                    payment.getCustomerId()
                )
            );
        }
    }
}
```

### 2. **Structured Context**

```java
public class StructuredContext {
    
    // Exception with structured context
    public class PaymentProcessingException extends BusinessException {
        
        private final String paymentId;
        private final String customerId;
        private final BigDecimal amount;
        private final String currency;
        private final PaymentStatus status;
        private final String gatewayResponse;
        
        private PaymentProcessingException(Builder builder) {
            super(builder.errorCode, builder.buildMessage());
            this.paymentId = builder.paymentId;
            this.customerId = builder.customerId;
            this.amount = builder.amount;
            this.currency = builder.currency;
            this.status = builder.status;
            this.gatewayResponse = builder.gatewayResponse;
            
            // Add structured context
            addContext("paymentId", paymentId);
            addContext("customerId", customerId);
            addContext("amount", amount);
            addContext("currency", currency);
            addContext("status", status);
            addContext("gatewayResponse", gatewayResponse);
        }
        
        public static Builder builder() {
            return new Builder();
        }
        
        public static class Builder {
            private ErrorCode errorCode;
            private String paymentId;
            private String customerId;
            private BigDecimal amount;
            private String currency;
            private PaymentStatus status;
            private String gatewayResponse;
            
            public Builder errorCode(ErrorCode errorCode) {
                this.errorCode = errorCode;
                return this;
            }
            
            public Builder paymentId(String paymentId) {
                this.paymentId = paymentId;
                return this;
            }
            
            public Builder customerId(String customerId) {
                this.customerId = customerId;
                return this;
            }
            
            public Builder amount(BigDecimal amount) {
                this.amount = amount;
                return this;
            }
            
            public Builder currency(String currency) {
                this.currency = currency;
                return this;
            }
            
            public Builder status(PaymentStatus status) {
                this.status = status;
                return this;
            }
            
            public Builder gatewayResponse(String gatewayResponse) {
                this.gatewayResponse = gatewayResponse;
                return this;
            }
            
            private String buildMessage() {
                return String.format(
                    "Payment processing failed. Payment ID: %s, Customer: %s, Amount: %s %s, Status: %s",
                    paymentId, customerId, amount, currency, status
                );
            }
            
            public PaymentProcessingException build() {
                Objects.requireNonNull(errorCode, "errorCode required");
                Objects.requireNonNull(paymentId, "paymentId required");
                return new PaymentProcessingException(this);
            }
        }
        
        // Getters for all fields...
    }
    
    // Usage
    throw PaymentProcessingException.builder()
        .errorCode(ErrorCode.PAYMENT_DECLINED)
        .paymentId("PAY-123")
        .customerId("CUST-456")
        .amount(new BigDecimal("99.99"))
        .currency("USD")
        .status(PaymentStatus.DECLINED)
        .gatewayResponse("Insufficient funds")
        .build();
}
```

### 3. **User-Friendly vs Technical Messages**

```java
public class DualMessages {
    
    public abstract class DualMessageException extends RuntimeException {
        
        private final String userMessage;  // For end users
        private final String technicalMessage;  // For logs/developers
        
        protected DualMessageException(String userMessage, String technicalMessage) {
            super(technicalMessage);
            this.userMessage = userMessage;
            this.technicalMessage = technicalMessage;
        }
        
        public String getUserMessage() {
            return userMessage;
        }
        
        public String getTechnicalMessage() {
            return technicalMessage;
        }
    }
    
    public class PaymentDeclinedException extends DualMessageException {
        public PaymentDeclinedException(String paymentId, String reason) {
            super(
                "Your payment could not be processed. Please try a different payment method.",
                String.format("Payment declined. Payment ID: %s, Reason: %s", paymentId, reason)
            );
        }
    }
    
    // Usage in REST controller
    @RestController
    public class PaymentController {
        
        @PostMapping("/payments")
        public ResponseEntity<PaymentResponse> processPayment(@RequestBody PaymentRequest request) {
            try {
                Payment payment = paymentService.processPayment(request);
                return ResponseEntity.ok(new PaymentResponse(payment));
            } catch (DualMessageException e) {
                logger.error(e.getTechnicalMessage(), e);
                return ResponseEntity
                    .badRequest()
                    .body(new ErrorResponse(e.getUserMessage()));
            }
        }
    }
}
```

---

## Exception Translation

### 1. **Layer-Specific Exception Translation**

```java
public class LayeredExceptionTranslation {
    
    // Database Layer → Repository Layer
    @Repository
    public class OrderRepository {
        
        public Order save(Order order) {
            try {
                return jdbcTemplate.update(
                    "INSERT INTO orders (id, customer_id, total) VALUES (?, ?, ?)",
                    order.getId(), order.getCustomerId(), order.getTotal()
                );
            } catch (DuplicateKeyException e) {
                throw new OrderAlreadyExistsException(order.getId(), e);
            } catch (DataIntegrityViolationException e) {
                throw new InvalidOrderDataException("Invalid order data", e)
                    .addContext("orderId", order.getId());
            } catch (DataAccessException e) {
                throw new DatabaseException("Failed to save order", e)
                    .addContext("orderId", order.getId());
            }
        }
    }
    
    // Repository Layer → Service Layer
    @Service
    public class OrderService {
        
        private final OrderRepository orderRepository;
        private final PaymentService paymentService;
        
        public Order createOrder(OrderRequest request) {
            try {
                Order order = new Order(request);
                Order savedOrder = orderRepository.save(order);
                
                paymentService.processPayment(savedOrder.getPaymentInfo());
                
                return savedOrder;
            } catch (PaymentDeclinedException e) {
                // Translate payment exception to order exception
                throw new OrderCreationFailedException(
                    "Order creation failed due to payment decline",
                    e
                ).addContext("orderId", order.getId());
            }
        }
    }
    
    // Service Layer → Controller Layer
    @RestController
    public class OrderController {
        
        private final OrderService orderService;
        
        @PostMapping("/orders")
        public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
            try {
                Order order = orderService.createOrder(request);
                return ResponseEntity.ok(new OrderResponse(order));
            } catch (OrderCreationFailedException e) {
                logger.error("Order creation failed", e);
                return ResponseEntity
                    .unprocessableEntity()
                    .body(new ErrorResponse(e.getMessage()));
            } catch (ValidationException e) {
                return ResponseEntity
                    .badRequest()
                    .body(new ErrorResponse(e.getMessage()));
            }
        }
    }
}
```

### 2. **Exception Wrapping**

```java
public class ExceptionWrapping {
    
    // ✅ GOOD - Wrap with cause
    public void goodWrapping() {
        try {
            externalApi.call();
        } catch (IOException e) {
            // Wrap low-level exception with high-level one
            throw new ExternalServiceException(
                "Failed to call external service",
                e  // Preserve original exception as cause
            );
        }
    }
    
    // ❌ BAD - Lose original exception
    public void badWrapping() {
        try {
            externalApi.call();
        } catch (IOException e) {
            throw new ExternalServiceException(
                "Failed to call external service"
                // Lost original exception! Can't debug root cause
            );
        }
    }
    
    // ✅ GOOD - Chain of exceptions preserves full context
    public void exceptionChain() {
        try {
            try {
                databaseCall();
            } catch (SQLException e) {
                throw new DatabaseException("DB call failed", e);
            }
        } catch (DatabaseException e) {
            throw new ServiceException("Service failed", e);
        }
    }
    
    // Unwrapping to find root cause
    public Throwable getRootCause(Throwable throwable) {
        Throwable cause = throwable;
        while (cause.getCause() != null) {
            cause = cause.getCause();
        }
        return cause;
    }
}
```

---

## Try-with-Resources

### 1. **Basic Usage**

```java
public class TryWithResources {
    
    // ❌ OLD WAY - Manual resource management
    public String readFileOld(String path) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader(path));
            return reader.readLine();
        } catch (IOException e) {
            throw new FileReadException("Failed to read file", e);
        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (IOException e) {
                    logger.warn("Failed to close reader", e);
                }
            }
        }
    }
    
    // ✅ NEW WAY - Automatic resource management
    public String readFile(String path) {
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            return reader.readLine();
        } catch (IOException e) {
            throw new FileReadException("Failed to read file", e);
        }
        // reader.close() called automatically, even if exception thrown
    }
    
    // Multiple resources
    public void copyFile(String source, String dest) {
        try (
            InputStream in = new FileInputStream(source);
            OutputStream out = new FileOutputStream(dest)
        ) {
            byte[] buffer = new byte[1024];
            int length;
            while ((length = in.read(buffer)) > 0) {
                out.write(buffer, 0, length);
            }
        } catch (IOException e) {
            throw new FileCopyException("Failed to copy file", e);
        }
        // Both streams closed automatically
    }
}
```

### 2. **Custom AutoCloseable Resources**

```java
public class CustomResources {
    
    // Custom resource that auto-closes
    public class DatabaseConnection implements AutoCloseable {
        
        private final Connection connection;
        
        public DatabaseConnection(String url) throws SQLException {
            this.connection = DriverManager.getConnection(url);
        }
        
        public ResultSet executeQuery(String sql) throws SQLException {
            return connection.createStatement().executeQuery(sql);
        }
        
        @Override
        public void close() {
            try {
                if (connection != null && !connection.isClosed()) {
                    connection.close();
                    logger.info("Database connection closed");
                }
            } catch (SQLException e) {
                logger.error("Failed to close database connection", e);
                // Don't throw from close() - could suppress other exceptions
            }
        }
    }
    
    // Usage
    public void useCustomResource() {
        try (DatabaseConnection db = new DatabaseConnection("jdbc:...")) {
            ResultSet rs = db.executeQuery("SELECT * FROM users");
            // Process results
        } catch (SQLException e) {
            throw new DatabaseException("Database operation failed", e);
        }
        // Connection closed automatically
    }
    
    // Distributed transaction example
    public class TransactionScope implements AutoCloseable {
        
        private final Transaction transaction;
        private boolean committed = false;
        
        public TransactionScope() {
            this.transaction = transactionManager.beginTransaction();
        }
        
        public void commit() {
            transaction.commit();
            committed = true;
        }
        
        @Override
        public void close() {
            if (!committed) {
                transaction.rollback();  // Auto-rollback if not committed
                logger.warn("Transaction rolled back");
            }
        }
    }
    
    // Usage
    public void performTransaction() {
        try (TransactionScope tx = new TransactionScope()) {
            // Perform database operations
            userRepository.save(user);
            orderRepository.save(order);
            
            tx.commit();  // Explicit commit
        }
        // If exception thrown before commit, transaction rolls back automatically
    }
}
```

---

## Async Exception Handling

### 1. **CompletableFuture Exception Handling**

```java
public class AsyncExceptionHandling {
    
    // Handle exceptions in CompletableFuture
    public CompletableFuture<User> getUser(String id) {
        return CompletableFuture.supplyAsync(() -> {
            User user = userRepository.findById(id);
            if (user == null) {
                throw new UserNotFoundException(id);
            }
            return user;
        })
        .exceptionally(ex -> {
            // Handle exception, return fallback
            logger.error("Failed to get user", ex);
            return User.guest();
        });
    }
    
    // handle() - process both success and failure
    public CompletableFuture<Result<User>> getUserWithResult(String id) {
        return CompletableFuture.supplyAsync(() -> userRepository.findById(id))
            .handle((user, ex) -> {
                if (ex != null) {
                    logger.error("Failed to get user", ex);
                    return Result.error(ex.getMessage());
                }
                return Result.success(user);
            });
    }
    
    // whenComplete() - side effects without changing result
    public CompletableFuture<User> getUserWithMetrics(String id) {
        return CompletableFuture.supplyAsync(() -> userRepository.findById(id))
            .whenComplete((user, ex) -> {
                if (ex != null) {
                    metricsService.incrementCounter("user.fetch.failure");
                    logger.error("Failed to get user", ex);
                } else {
                    metricsService.incrementCounter("user.fetch.success");
                }
            });
    }
    
    // Combining multiple futures with error handling
    public CompletableFuture<OrderSummary> getOrderSummary(String orderId) {
        CompletableFuture<Order> orderFuture = getOrder(orderId)
            .exceptionally(ex -> {
                logger.error("Failed to get order", ex);
                return Order.empty();
            });
        
        CompletableFuture<User> userFuture = getUser(orderId)
            .exceptionally(ex -> {
                logger.error("Failed to get user", ex);
                return User.guest();
            });
        
        return orderFuture.thenCombine(userFuture,
            (order, user) -> new OrderSummary(order, user)
        );
    }
}
```

### 2. **Reactive Streams Exception Handling**

```java
@Service
public class ReactiveExceptionHandling {
    
    // onErrorReturn - return fallback value
    public Mono<User> getUser(String id) {
        return userRepository.findById(id)
            .onErrorReturn(throwable -> {
                logger.error("Failed to get user", throwable);
                return User.guest();
            });
    }
    
    // onErrorResume - switch to fallback publisher
    public Mono<User> getUserWithFallback(String id) {
        return primaryRepository.findById(id)
            .onErrorResume(ex -> {
                logger.warn("Primary repository failed, trying secondary", ex);
                return secondaryRepository.findById(id);
            })
            .onErrorResume(ex -> {
                logger.error("Both repositories failed", ex);
                return Mono.just(User.guest());
            });
    }
    
    // onErrorMap - transform exception
    public Mono<User> getUserMapped(String id) {
        return userRepository.findById(id)
            .onErrorMap(SQLException.class, ex -> 
                new DatabaseException("Failed to fetch user", ex)
            );
    }
    
    // doOnError - side effect without changing stream
    public Mono<User> getUserWithLogging(String id) {
        return userRepository.findById(id)
            .doOnError(ex -> {
                logger.error("Failed to get user", ex);
                metricsService.incrementCounter("user.fetch.failure");
            });
    }
    
    // retry - with exponential backoff
    public Mono<User> getUserWithRetry(String id) {
        return userRepository.findById(id)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(throwable -> throwable instanceof TransientException)
                .doBeforeRetry(retrySignal -> 
                    logger.warn("Retrying, attempt: {}", retrySignal.totalRetries())
                )
            );
    }
    
    // timeout with fallback
    public Mono<User> getUserWithTimeout(String id) {
        return userRepository.findById(id)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(TimeoutException.class, ex -> {
                logger.error("Request timed out", ex);
                return Mono.just(User.fromCache(id));
            });
    }
}
```

---

## Logging and Monitoring

### 1. **Structured Logging**

```java
public class StructuredLogging {
    
    private static final Logger logger = LoggerFactory.getLogger(StructuredLogging.class);
    
    // ❌ BAD - Unstructured logging
    public void badLogging(Order order) {
        try {
            processOrder(order);
        } catch (Exception e) {
            logger.error("Error processing order: " + e.getMessage());
        }
    }
    
    // ✅ GOOD - Structured logging with context
    public void goodLogging(Order order) {
        try {
            processOrder(order);
        } catch (OrderProcessingException e) {
            logger.error(
                "Order processing failed. OrderId: {}, CustomerId: {}, ErrorCode: {}, Message: {}",
                e.getOrderId(),
                e.getCustomerId(),
                e.getErrorCode(),
                e.getMessage(),
                e  // Include full exception with stack trace
            );
        }
    }
    
    // ✅ BETTER - Use MDC (Mapped Diagnostic Context)
    public void mdcLogging(Order order) {
        MDC.put("orderId", order.getId());
        MDC.put("customerId", order.getCustomerId());
        
        try {
            processOrder(order);
        } catch (OrderProcessingException e) {
            logger.error(
                "Order processing failed. ErrorCode: {}, Message: {}",
                e.getErrorCode(),
                e.getMessage(),
                e
            );
            // orderId and customerId automatically included in log output
        } finally {
            MDC.clear();  // Always clear MDC
        }
    }
    
    // ✅ BEST - Structured logging with JSON
    public void jsonLogging(Order order) {
        try {
            processOrder(order);
        } catch (OrderProcessingException e) {
            Map<String, Object> logData = Map.of(
                "eventType", "ORDER_PROCESSING_FAILED",
                "orderId", e.getOrderId(),
                "customerId", e.getCustomerId(),
                "errorCode", e.getErrorCode().getCode(),
                "message", e.getMessage(),
                "context", e.getContext(),
                "timestamp", Instant.now()
            );
            
            logger.error("Order processing failed: {}", 
                objectMapper.writeValueAsString(logData), e);
        }
    }
}
```

### 2. **Metrics and Alerting**

```java
@Service
public class ExceptionMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public ExceptionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    // Count exceptions by type
    public void recordException(Exception e) {
        Counter.builder("exceptions")
            .tag("type", e.getClass().getSimpleName())
            .tag("retryable", String.valueOf(isRetryable(e)))
            .register(meterRegistry)
            .increment();
    }
    
    // Track error rate
    public void trackOperation(String operation) {
        try {
            performOperation(operation);
            
            Counter.builder("operations")
                .tag("name", operation)
                .tag("status", "success")
                .register(meterRegistry)
                .increment();
                
        } catch (Exception e) {
            Counter.builder("operations")
                .tag("name", operation)
                .tag("status", "failure")
                .tag("errorType", e.getClass().getSimpleName())
                .register(meterRegistry)
                .increment();
            
            throw e;
        }
    }
    
    // Alert on critical exceptions
    public void processPayment(Payment payment) {
        try {
            paymentGateway.charge(payment);
        } catch (PaymentGatewayException e) {
            // Record metric
            recordException(e);
            
            // Send alert for infrastructure failures
            if (e.isRetryable()) {
                alertService.sendAlert(
                    "Payment Gateway Error",
                    String.format("Payment gateway unavailable. Payment ID: %s", 
                        payment.getId())
                );
            }
            
            throw e;
        }
    }
}
```

---

## Testing Exceptions

### 1. **JUnit 5 Exception Testing**

```java
public class ExceptionTesting {
    
    private UserService userService;
    
    @BeforeEach
    void setUp() {
        userService = new UserService(userRepository);
    }
    
    // Test that exception is thrown
    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        // Arrange
        String userId = "nonexistent";
        when(userRepository.findById(userId)).thenReturn(Optional.empty());
        
        // Act & Assert
        assertThrows(UserNotFoundException.class, () -> {
            userService.getUser(userId);
        });
    }
    
    // Test exception message
    @Test
    void shouldThrowExceptionWithCorrectMessage() {
        String userId = "123";
        
        UserNotFoundException exception = assertThrows(
            UserNotFoundException.class,
            () -> userService.getUser(userId)
        );
        
        assertEquals("User not found: 123", exception.getMessage());
    }
    
    // Test exception details
    @Test
    void shouldThrowExceptionWithCorrectContext() {
        String userId = "123";
        
        UserNotFoundException exception = assertThrows(
            UserNotFoundException.class,
            () -> userService.getUser(userId)
        );
        
        assertEquals(userId, exception.getContext().get("userId"));
        assertEquals(ErrorCode.USER_NOT_FOUND, exception.getErrorCode());
        assertEquals(404, exception.getHttpStatus());
    }
    
    // Test exception cause
    @Test
    void shouldWrapDatabaseException() {
        when(userRepository.findById(any()))
            .thenThrow(new DataAccessException("DB error"));
        
        DatabaseException exception = assertThrows(
            DatabaseException.class,
            () -> userService.getUser("123")
        );
        
        assertInstanceOf(DataAccessException.class, exception.getCause());
        assertTrue(exception.isRetryable());
    }
    
    // Test no exception thrown
    @Test
    void shouldNotThrowExceptionWhenUserExists() {
        User user = new User("123", "John");
        when(userRepository.findById("123")).thenReturn(Optional.of(user));
        
        assertDoesNotThrow(() -> userService.getUser("123"));
    }
}
```

### 2. **Testing Exception Handling**

```java
public class ExceptionHandlingTest {
    
    @Test
    void shouldHandleExceptionAndReturnFallback() {
        // Arrange
        when(primaryService.getData()).thenThrow(new ServiceException("Error"));
        when(fallbackService.getData()).thenReturn("fallback data");
        
        // Act
        String result = resilientService.getData();
        
        // Assert
        assertEquals("fallback data", result);
        verify(primaryService).getData();
        verify(fallbackService).getData();
    }
    
    @Test
    void shouldRetryOnTransientException() {
        // Arrange - fail twice, succeed third time
        when(externalApi.call())
            .thenThrow(new TransientException("Timeout"))
            .thenThrow(new TransientException("Timeout"))
            .thenReturn("success");
        
        // Act
        String result = retryableService.callWithRetry();
        
        // Assert
        assertEquals("success", result);
        verify(externalApi, times(3)).call();
    }
    
    @Test
    void shouldNotRetryOnPermanentException() {
        // Arrange
        when(externalApi.call())
            .thenThrow(new PermanentException("Invalid request"));
        
        // Act & Assert
        assertThrows(PermanentException.class, () -> 
            retryableService.callWithRetry()
        );
        
        verify(externalApi, times(1)).call();  // No retry
    }
}
```

---

## Anti-Patterns to Avoid

### 1. **Exception Anti-Patterns**

```java
public class ExceptionAntiPatterns {
    
    // ❌ ANTI-PATTERN 1: Swallowing exceptions
    public void swallowingException() {
        try {
            riskyOperation();
        } catch (Exception e) {
            // Silent failure - very bad!
        }
    }
    
    // ❌ ANTI-PATTERN 2: Catching Exception/Throwable
    public void catchingEverything() {
        try {
            operation();
        } catch (Throwable t) {  // Catches OutOfMemoryError, etc.
            // Can't recover from these!
        }
    }
    
    // ❌ ANTI-PATTERN 3: Using exceptions for flow control
    public boolean isNumeric(String str) {
        try {
            Integer.parseInt(str);
            return true;
        } catch (NumberFormatException e) {
            return false;  // Expensive!
        }
    }
    
    // ✅ CORRECT
    public boolean isNumericCorrect(String str) {
        return str != null && str.matches("-?\\d+");
    }
    
    // ❌ ANTI-PATTERN 4: Throwing generic exceptions
    public void genericException() throws Exception {  // Too vague!
        throw new Exception("Something went wrong");
    }
    
    // ✅ CORRECT
    public void specificException() {
        throw new InvalidOrderStateException("Order cannot be modified");
    }
    
    // ❌ ANTI-PATTERN 5: Logging and rethrowing
    public void logAndRethrow() {
        try {
            operation();
        } catch (Exception e) {
            logger.error("Error", e);  // Logged here
            throw e;  // Will be logged again higher up!
        }
    }
    
    // ✅ CORRECT - Log at appropriate level only
    public void logAtBoundary() {
        operation();  // Let exception propagate
    }
    
    // ❌ ANTI-PATTERN 6: Catching and wrapping without adding value
    public void uselessWrapping() {
        try {
            operation();
        } catch (IOException e) {
            throw new RuntimeException(e);  // Adds nothing!
        }
    }
    
    // ✅ CORRECT - Add context when wrapping
    public void valuableWrapping(String filename) {
        try {
            readFile(filename);
        } catch (IOException e) {
            throw new FileProcessingException(
                "Failed to process file: " + filename,
                e
            ).addContext("filename", filename);
        }
    }
    
    // ❌ ANTI-PATTERN 7: Returning null on error
    public User getUserBad(String id) {
        try {
            return userRepository.findById(id);
        } catch (Exception e) {
            logger.error("Error", e);
            return null;  // Swallows exception info!
        }
    }
    
    // ✅ CORRECT - Let exception propagate or use Optional
    public Optional<User> getUserGood(String id) {
        return userRepository.findById(id);
    }
    
    // ❌ ANTI-PATTERN 8: Exception in finally
    public void exceptionInFinally() {
        try {
            operation();
        } finally {
            cleanup();  // If this throws, original exception is lost!
        }
    }
    
    // ✅ CORRECT - Try-with-resources or catch in finally
    public void safeFinally() {
        try {
            operation();
        } finally {
            try {
                cleanup();
            } catch (Exception e) {
                logger.warn("Cleanup failed", e);
            }
        }
    }
}
```

---

## Framework-Specific Handling

### 1. **Spring Boot Exception Handling**

```java
// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    // Handle specific domain exceptions
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        logger.warn("User not found: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().getCode(),
            ex.getMessage(),
            ex.getContext()
        );
        
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(error);
    }
    
    // Handle validation exceptions
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        
        List<FieldError> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fieldError -> new FieldError(
                fieldError.getField(),
                fieldError.getDefaultMessage()
            ))
            .collect(Collectors.toList());
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            "VALIDATION_ERROR",
            "Validation failed",
            errors
        );
        
        return ResponseEntity.badRequest().body(response);
    }
    
    // Handle all business exceptions
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        logger.error("Business exception: {}", ex.getMessage(), ex);
        
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().getCode(),
            ex.getMessage(),
            ex.getContext()
        );
        
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(error);
    }
    
    // Handle infrastructure exceptions (retryable)
    @ExceptionHandler(InfrastructureException.class)
    public ResponseEntity<ErrorResponse> handleInfrastructureException(
            InfrastructureException ex) {
        
        logger.error("Infrastructure exception: {}", ex.getMessage(), ex);
        
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().getCode(),
            "Service temporarily unavailable. Please try again.",
            Map.of("retryable", true)
        );
        
        return ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "60")
            .body(error);
    }
    
    // Catch-all for unexpected exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        logger.error("Unexpected exception", ex);
        
        ErrorResponse error = new ErrorResponse(
            "INTERNAL_ERROR",
            "An unexpected error occurred",
            Collections.emptyMap()
        );
        
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(error);
    }
}

// Error response DTOs
record ErrorResponse(
    String errorCode,
    String message,
    Map<String, Object> context
) {}

record ValidationErrorResponse(
    String errorCode,
    String message,
    List<FieldError> errors
) {}

record FieldError(
    String field,
    String message
) {}
```

### 2. **Spring Transaction Exception Handling**

```java
@Service
public class TransactionalExceptionHandling {
    
    // Rollback on specific exceptions
    @Transactional(rollbackFor = {BusinessException.class})
    public void processOrder(Order order) {
        orderRepository.save(order);
        
        if (inventory.isInsufficient(order)) {
            throw new InsufficientInventoryException(
                order.getProductId(),
                order.getQuantity(),
                inventory.getAvailable()
            );
            // Transaction will rollback
        }
        
        paymentService.processPayment(order);
    }
    
    // Don't rollback on specific exceptions
    @Transactional(noRollbackFor = {ValidationException.class})
    public void processWithValidation(Order order) {
        // Transaction won't rollback for validation errors
        validateOrder(order);
        orderRepository.save(order);
    }
    
    // Handle exceptions with compensation
    @Transactional
    public void processOrderWithCompensation(Order order) {
        try {
            orderRepository.save(order);
            inventoryService.reserve(order);
            paymentService.charge(order);
        } catch (PaymentException e) {
            // Compensate for inventory reservation
            inventoryService.release(order);
            throw new OrderProcessingException("Payment failed", e);
            // Transaction still rolls back
        }
    }
}
```

---

## Decision Tree

```
Exception Design Decision Tree:

START: Need to signal an error?

├─ Is this a programming bug (null, invalid state, illegal args)?
│  └─► Use UNCHECKED exception
│      Examples: IllegalArgumentException, IllegalStateException, NullPointerException
│
├─ Is this a recoverable condition?
│  │
│  ├─ Can caller reasonably handle it?
│  │  │
│  │  ├─ Is it common/expected?
│  │  │  └─► Consider Optional, Result type, or boolean return
│  │  │      Don't use exceptions for flow control
│  │  │
│  │  └─ Is it rare/exceptional?
│  │     └─► Use UNCHECKED exception (modern approach)
│  │         Create domain-specific exception hierarchy
│  │
│  └─ External resource failure (network, file, DB)?
│     └─► Use UNCHECKED exception
│         Translate to domain exception at boundary
│
├─ At what layer am I?
│  │
│  ├─ Infrastructure/Repository Layer
│  │  └─► Translate tech exceptions to domain exceptions
│  │      (SQLException → DatabaseException)
│  │
│  ├─ Service Layer
│  │  └─► Use domain exceptions only
│  │      (UserNotFoundException, InsufficientInventoryException)
│  │
│  └─ Controller/API Layer
│  │  └─► Handle exceptions, return appropriate HTTP status
│  │      Use @ExceptionHandler or global handler
│  │
│  └─ Should I catch here?
│     │
│     ├─ Can I add meaningful context?
│     │  └─► YES: Catch, add context, rethrow
│     │
│     ├─ Can I recover/fallback?
│     │  └─► YES: Catch and handle
│     │
│     ├─ Need to log at this level?
│     │  └─► YES: Catch, log, rethrow (avoid duplicate logging)
│     │
│     └─ Otherwise
│        └─► NO: Let it propagate
│
└─ Exception already exists from framework?
   │
   ├─ Adds value to wrap it?
   │  └─► YES: Wrap with domain exception, preserve cause
   │
   └─ Otherwise
      └─► NO: Let it propagate or translate at boundary
```

### Quick Reference Matrix

| Scenario | Exception Type | Example |
|----------|---------------|---------|
| Null parameter | `IllegalArgumentException` | "Parameter 'userId' cannot be null" |
| Invalid state | `IllegalStateException` | "Cannot ship order in status: PENDING" |
| Resource not found | Custom unchecked | `UserNotFoundException` |
| Validation failure | Custom unchecked | `InvalidEmailException` |
| Business rule violation | Custom unchecked | `InsufficientInventoryException` |
| External service failure | Custom unchecked | `PaymentGatewayException` |
| Database error | Custom unchecked | `DatabaseException` |
| Configuration error | Custom unchecked | `ConfigurationException` |

---

## Best Practices Summary

### ✅ DO

1. **Use unchecked exceptions** for most cases (modern best practice)
2. **Create domain-specific hierarchy** with base exception and categories
3. **Include rich context** - error codes, structured data, user/tech messages
4. **Preserve exception chain** - always pass cause when wrapping
5. **Translate at boundaries** - infrastructure → domain → API response
6. **Log at appropriate level** - usually at boundaries only
7. **Use try-with-resources** for automatic cleanup
8. **Test exception scenarios** thoroughly
9. **Make messages actionable** - what happened, why, what to do
10. **Monitor and alert** on critical exceptions

### ❌ DON'T

1. **Don't swallow exceptions** - always log or rethrow
2. **Don't catch `Exception` or `Throwable`** - be specific
3. **Don't use exceptions for flow control** - expensive
4. **Don't log and rethrow** - causes duplicate logging
5. **Don't throw generic exceptions** - create specific ones
6. **Don't lose original cause** - preserve stack trace
7. **Don't return null on error** - use Optional or throw
8. **Don't throw from `finally`** - can hide original exception
9. **Don't catch exceptions you can't handle** - let them propagate
10. **Don't forget to clean up resources** - use try-with-resources

---

## Conclusion

Effective exception design requires:

1. **Well-designed hierarchy** - Domain-specific, meaningful categories
2. **Rich context** - Error codes, structured data, actionable messages
3. **Proper handling** - Catch at right level, translate appropriately
4. **Good logging** - Structured, with context, at boundaries
5. **Testing** - Verify exceptions thrown and handled correctly

**Modern approach**: Favor unchecked exceptions, rich context, clear domain model, handle at boundaries.

**Remember**: Exceptions are for exceptional conditions. Use Optional, Result types, or boolean returns for expected conditions.
