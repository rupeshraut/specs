# Effective Modern Java Enums

A comprehensive guide to mastering Java enums, advanced patterns, best practices, and real-world applications.

---

## Table of Contents

1. [Enum Fundamentals](#enum-fundamentals)
2. [Enum with Fields and Methods](#enum-with-fields-and-methods)
3. [Enum with Constructors](#enum-with-constructors)
4. [Enum Implementing Interfaces](#enum-implementing-interfaces)
5. [Abstract Methods in Enums](#abstract-methods-in-enums)
6. [EnumSet and EnumMap](#enumset-and-enummap)
7. [Enum-Based Singleton](#enum-based-singleton)
8. [State Machine Pattern](#state-machine-pattern)
9. [Strategy Pattern with Enums](#strategy-pattern-with-enums)
10. [Command Pattern with Enums](#command-pattern-with-enums)
11. [Enum for Configuration](#enum-for-configuration)
12. [Enum Best Practices](#enum-best-practices)
13. [Common Pitfalls](#common-pitfalls)

---

## Enum Fundamentals

### Basic Enum

```java
// Simple enum
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

// Usage
Day today = Day.MONDAY;

// Built-in methods
String name = today.name();           // "MONDAY"
int ordinal = today.ordinal();        // 0
Day[] values = Day.values();          // All enum constants
Day parsed = Day.valueOf("MONDAY");   // Parse from string

// Comparison
if (today == Day.MONDAY) { }          // Reference equality (preferred)
if (today.equals(Day.MONDAY)) { }     // Works but unnecessary

// Switch statement
switch (today) {
    case MONDAY:
        System.out.println("Start of work week");
        break;
    case FRIDAY:
        System.out.println("Almost weekend!");
        break;
    case SATURDAY:
    case SUNDAY:
        System.out.println("Weekend!");
        break;
    default:
        System.out.println("Midweek");
}

// Enhanced switch (Java 14+)
String message = switch (today) {
    case MONDAY -> "Start of work week";
    case FRIDAY -> "Almost weekend!";
    case SATURDAY, SUNDAY -> "Weekend!";
    default -> "Midweek";
};
```

### Why Use Enums?

```java
// ❌ BAD: Using constants
public class Status {
    public static final int PENDING = 0;
    public static final int APPROVED = 1;
    public static final int REJECTED = 2;
}

int status = Status.PENDING;
status = 999;  // No type safety!
if (status == Status.PENDING) { }  // Magic numbers

// ✅ GOOD: Using enum
public enum Status {
    PENDING, APPROVED, REJECTED
}

Status status = Status.PENDING;
// status = 999;  // Compile error!
if (status == Status.PENDING) { }  // Type-safe
```

**Benefits:**
- **Type Safety**: Compile-time checking
- **Readability**: Self-documenting code
- **Namespace**: No constant collision
- **Methods**: Can have behavior
- **Iteration**: Easy to iterate all values

---

## Enum with Fields and Methods

### Adding Fields

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6),
    JUPITER(1.9e+27, 7.1492e7),
    SATURN(5.688e+26, 6.0268e7),
    URANUS(8.686e+25, 2.5559e7),
    NEPTUNE(1.024e+26, 2.4746e7);
    
    private final double mass;   // in kilograms
    private final double radius; // in meters
    
    // Universal gravitational constant (m^3 kg^-1 s^-2)
    private static final double G = 6.67300E-11;
    
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }
    
    public double mass() { return mass; }
    public double radius() { return radius; }
    
    // Calculate surface gravity
    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }
    
    // Calculate weight on planet
    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}

// Usage
double earthWeight = 75.0; // kg
for (Planet p : Planet.values()) {
    System.out.printf("Weight on %s is %.2f N%n",
        p, p.surfaceWeight(earthWeight));
}
```

### Enum with Business Logic

```java
public enum OrderStatus {
    PENDING("Order received, awaiting processing"),
    PROCESSING("Order is being processed"),
    SHIPPED("Order has been shipped"),
    DELIVERED("Order delivered successfully"),
    CANCELLED("Order cancelled");
    
    private final String description;
    
    OrderStatus(String description) {
        this.description = description;
    }
    
    public String getDescription() {
        return description;
    }
    
    public boolean canTransitionTo(OrderStatus newStatus) {
        return switch (this) {
            case PENDING -> newStatus == PROCESSING || newStatus == CANCELLED;
            case PROCESSING -> newStatus == SHIPPED || newStatus == CANCELLED;
            case SHIPPED -> newStatus == DELIVERED;
            case DELIVERED, CANCELLED -> false;
        };
    }
    
    public boolean isFinal() {
        return this == DELIVERED || this == CANCELLED;
    }
    
    public boolean isActive() {
        return !isFinal();
    }
}

// Usage
OrderStatus status = OrderStatus.PENDING;

if (status.canTransitionTo(OrderStatus.PROCESSING)) {
    status = OrderStatus.PROCESSING;
}

String description = status.getDescription();
boolean canChange = status.isActive();
```

---

## Enum with Constructors

### Multiple Constructors

```java
public enum HttpStatus {
    // Informational
    CONTINUE(100, "Continue"),
    SWITCHING_PROTOCOLS(101, "Switching Protocols"),
    
    // Success
    OK(200, "OK"),
    CREATED(201, "Created"),
    ACCEPTED(202, "Accepted"),
    NO_CONTENT(204, "No Content"),
    
    // Redirection
    MOVED_PERMANENTLY(301, "Moved Permanently"),
    FOUND(302, "Found"),
    NOT_MODIFIED(304, "Not Modified"),
    
    // Client Error
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    
    // Server Error
    INTERNAL_SERVER_ERROR(500, "Internal Server Error"),
    BAD_GATEWAY(502, "Bad Gateway"),
    SERVICE_UNAVAILABLE(503, "Service Unavailable");
    
    private final int code;
    private final String reasonPhrase;
    
    HttpStatus(int code, String reasonPhrase) {
        this.code = code;
        this.reasonPhrase = reasonPhrase;
    }
    
    public int getCode() { return code; }
    public String getReasonPhrase() { return reasonPhrase; }
    
    public boolean isInformational() { return code >= 100 && code < 200; }
    public boolean isSuccess() { return code >= 200 && code < 300; }
    public boolean isRedirection() { return code >= 300 && code < 400; }
    public boolean isClientError() { return code >= 400 && code < 500; }
    public boolean isServerError() { return code >= 500 && code < 600; }
    public boolean isError() { return isClientError() || isServerError(); }
    
    // Reverse lookup
    public static HttpStatus valueOf(int code) {
        for (HttpStatus status : values()) {
            if (status.code == code) {
                return status;
            }
        }
        throw new IllegalArgumentException("Unknown status code: " + code);
    }
}

// Usage
HttpStatus status = HttpStatus.OK;
System.out.println(status.getCode() + " " + status.getReasonPhrase());

if (status.isSuccess()) {
    System.out.println("Request succeeded");
}

// Reverse lookup
HttpStatus fromCode = HttpStatus.valueOf(404);
System.out.println(fromCode);  // NOT_FOUND
```

---

## Enum Implementing Interfaces

### Single Interface

```java
public interface Describable {
    String getDescription();
}

public enum Priority implements Describable {
    LOW("Low priority - can wait"),
    MEDIUM("Medium priority - should be done soon"),
    HIGH("High priority - do it now"),
    CRITICAL("Critical - drop everything!");
    
    private final String description;
    
    Priority(String description) {
        this.description = description;
    }
    
    @Override
    public String getDescription() {
        return description;
    }
    
    public boolean isUrgent() {
        return this == HIGH || this == CRITICAL;
    }
}

// Usage
Priority priority = Priority.HIGH;
System.out.println(priority.getDescription());

if (priority.isUrgent()) {
    handleUrgently();
}
```

### Multiple Interfaces

```java
public interface Weighted {
    int getWeight();
}

public interface Colored {
    String getColor();
}

public enum Severity implements Weighted, Colored {
    INFO(1, "blue"),
    WARNING(5, "yellow"),
    ERROR(10, "orange"),
    CRITICAL(20, "red");
    
    private final int weight;
    private final String color;
    
    Severity(int weight, String color) {
        this.weight = weight;
        this.color = color;
    }
    
    @Override
    public int getWeight() { return weight; }
    
    @Override
    public String getColor() { return color; }
    
    public boolean isMoreSevereThan(Severity other) {
        return this.weight > other.weight;
    }
}

// Usage
Severity sev1 = Severity.ERROR;
Severity sev2 = Severity.WARNING;

if (sev1.isMoreSevereThan(sev2)) {
    System.out.println("Error is more severe than warning");
}
```

---

## Abstract Methods in Enums

### Constant-Specific Method Implementation

```java
public enum Operation {
    PLUS("+") {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS("-") {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    },
    TIMES("*") {
        @Override
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE("/") {
        @Override
        public double apply(double x, double y) {
            if (y == 0) {
                throw new ArithmeticException("Division by zero");
            }
            return x / y;
        }
    };
    
    private final String symbol;
    
    Operation(String symbol) {
        this.symbol = symbol;
    }
    
    public String getSymbol() { return symbol; }
    
    // Abstract method - each constant must implement
    public abstract double apply(double x, double y);
    
    @Override
    public String toString() {
        return symbol;
    }
}

// Usage
double result = Operation.PLUS.apply(2, 3);      // 5.0
double product = Operation.TIMES.apply(4, 5);    // 20.0

// Dynamic calculation
public double calculate(double x, double y, Operation op) {
    return op.apply(x, y);
}

// Calculator
for (Operation op : Operation.values()) {
    System.out.printf("2 %s 3 = %.2f%n", op, op.apply(2, 3));
}
```

### Strategy Pattern with Abstract Methods

```java
public enum PaymentStrategy {
    CREDIT_CARD {
        @Override
        public void pay(double amount) {
            System.out.println("Processing credit card payment: $" + amount);
            // Credit card processing logic
        }
        
        @Override
        public double calculateFee(double amount) {
            return amount * 0.029 + 0.30;  // 2.9% + $0.30
        }
    },
    
    PAYPAL {
        @Override
        public void pay(double amount) {
            System.out.println("Processing PayPal payment: $" + amount);
            // PayPal processing logic
        }
        
        @Override
        public double calculateFee(double amount) {
            return amount * 0.034 + 0.30;  // 3.4% + $0.30
        }
    },
    
    BANK_TRANSFER {
        @Override
        public void pay(double amount) {
            System.out.println("Processing bank transfer: $" + amount);
            // Bank transfer logic
        }
        
        @Override
        public double calculateFee(double amount) {
            return 5.00;  // Flat fee
        }
    },
    
    CRYPTOCURRENCY {
        @Override
        public void pay(double amount) {
            System.out.println("Processing crypto payment: $" + amount);
            // Crypto processing logic
        }
        
        @Override
        public double calculateFee(double amount) {
            return amount * 0.01;  // 1%
        }
    };
    
    public abstract void pay(double amount);
    public abstract double calculateFee(double amount);
    
    public double getTotalAmount(double amount) {
        return amount + calculateFee(amount);
    }
}

// Usage
PaymentStrategy strategy = PaymentStrategy.CREDIT_CARD;
double amount = 100.0;

strategy.pay(amount);
double fee = strategy.calculateFee(amount);
double total = strategy.getTotalAmount(amount);

System.out.printf("Amount: $%.2f, Fee: $%.2f, Total: $%.2f%n", 
    amount, fee, total);
```

---

## EnumSet and EnumMap

### EnumSet - Efficient Set Implementation

```java
public enum Permission {
    READ, WRITE, EXECUTE, DELETE
}

// Create EnumSet
EnumSet<Permission> userPermissions = EnumSet.of(Permission.READ, Permission.WRITE);
EnumSet<Permission> adminPermissions = EnumSet.allOf(Permission.class);
EnumSet<Permission> readOnly = EnumSet.of(Permission.READ);
EnumSet<Permission> empty = EnumSet.noneOf(Permission.class);

// Set operations (very efficient - uses bit vector internally)
EnumSet<Permission> permissions = EnumSet.of(Permission.READ, Permission.WRITE);

permissions.add(Permission.EXECUTE);
permissions.remove(Permission.WRITE);

boolean canRead = permissions.contains(Permission.READ);
boolean isEmpty = permissions.isEmpty();

// Bulk operations
EnumSet<Permission> additional = EnumSet.of(Permission.DELETE);
permissions.addAll(additional);

EnumSet<Permission> readWrite = EnumSet.of(Permission.READ, Permission.WRITE);
permissions.retainAll(readWrite);

// Complement
EnumSet<Permission> notAllowed = EnumSet.complementOf(permissions);

// Range
EnumSet<Permission> range = EnumSet.range(Permission.READ, Permission.EXECUTE);
```

### EnumMap - Efficient Map Implementation

```java
public enum Status {
    PENDING, PROCESSING, COMPLETED, FAILED
}

// Create EnumMap
EnumMap<Status, Integer> statusCounts = new EnumMap<>(Status.class);

// Populate
statusCounts.put(Status.PENDING, 10);
statusCounts.put(Status.PROCESSING, 5);
statusCounts.put(Status.COMPLETED, 100);
statusCounts.put(Status.FAILED, 2);

// Access
int pendingCount = statusCounts.get(Status.PENDING);
int completedCount = statusCounts.getOrDefault(Status.COMPLETED, 0);

// Iterate
for (Map.Entry<Status, Integer> entry : statusCounts.entrySet()) {
    System.out.printf("%s: %d%n", entry.getKey(), entry.getValue());
}

// Real-world example: State machine with actions
public class OrderProcessor {
    private final EnumMap<OrderStatus, Consumer<Order>> handlers = 
        new EnumMap<>(OrderStatus.class);
    
    public OrderProcessor() {
        handlers.put(OrderStatus.PENDING, this::handlePending);
        handlers.put(OrderStatus.PROCESSING, this::handleProcessing);
        handlers.put(OrderStatus.SHIPPED, this::handleShipped);
        handlers.put(OrderStatus.DELIVERED, this::handleDelivered);
    }
    
    public void process(Order order) {
        Consumer<Order> handler = handlers.get(order.getStatus());
        if (handler != null) {
            handler.accept(order);
        }
    }
    
    private void handlePending(Order order) {
        // Handle pending order
    }
    
    private void handleProcessing(Order order) {
        // Handle processing order
    }
    
    private void handleShipped(Order order) {
        // Handle shipped order
    }
    
    private void handleDelivered(Order order) {
        // Handle delivered order
    }
}
```

---

## Enum-Based Singleton

### Thread-Safe Singleton

```java
// ✅ Best way to implement singleton in Java
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    // Constructor called only once
    DatabaseConnection() {
        try {
            connection = DriverManager.getConnection(
                "jdbc:postgresql://localhost/mydb",
                "user",
                "password"
            );
            System.out.println("Database connected");
        } catch (SQLException e) {
            throw new RuntimeException("Failed to connect", e);
        }
    }
    
    public Connection getConnection() {
        return connection;
    }
    
    public void executeQuery(String sql) {
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            // Process results
        } catch (SQLException e) {
            throw new RuntimeException("Query failed", e);
        }
    }
}

// Usage
DatabaseConnection.INSTANCE.executeQuery("SELECT * FROM users");
Connection conn = DatabaseConnection.INSTANCE.getConnection();

// Benefits:
// 1. Thread-safe by default
// 2. Serialization-safe
// 3. Cannot be instantiated via reflection
// 4. Lazy initialization (first access)
// 5. Simple and concise
```

### Configuration Singleton

```java
public enum AppConfig {
    INSTANCE;
    
    private final Properties properties = new Properties();
    
    AppConfig() {
        try (InputStream input = getClass()
                .getResourceAsStream("/application.properties")) {
            properties.load(input);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load config", e);
        }
    }
    
    public String get(String key) {
        return properties.getProperty(key);
    }
    
    public String get(String key, String defaultValue) {
        return properties.getProperty(key, defaultValue);
    }
    
    public int getInt(String key) {
        return Integer.parseInt(get(key));
    }
    
    public boolean getBoolean(String key) {
        return Boolean.parseBoolean(get(key));
    }
}

// Usage
String dbUrl = AppConfig.INSTANCE.get("database.url");
int port = AppConfig.INSTANCE.getInt("server.port");
boolean debugMode = AppConfig.INSTANCE.getBoolean("debug.enabled");
```

---

## State Machine Pattern

### Order State Machine

```java
public enum OrderState {
    CREATED {
        @Override
        public OrderState next() {
            return PENDING_PAYMENT;
        }
        
        @Override
        public boolean canCancel() {
            return true;
        }
    },
    
    PENDING_PAYMENT {
        @Override
        public OrderState next() {
            return PAYMENT_CONFIRMED;
        }
        
        @Override
        public boolean canCancel() {
            return true;
        }
    },
    
    PAYMENT_CONFIRMED {
        @Override
        public OrderState next() {
            return PROCESSING;
        }
        
        @Override
        public boolean canCancel() {
            return true;
        }
    },
    
    PROCESSING {
        @Override
        public OrderState next() {
            return SHIPPED;
        }
        
        @Override
        public boolean canCancel() {
            return false;
        }
    },
    
    SHIPPED {
        @Override
        public OrderState next() {
            return DELIVERED;
        }
        
        @Override
        public boolean canCancel() {
            return false;
        }
    },
    
    DELIVERED {
        @Override
        public OrderState next() {
            throw new IllegalStateException("Order already delivered");
        }
        
        @Override
        public boolean canCancel() {
            return false;
        }
    },
    
    CANCELLED {
        @Override
        public OrderState next() {
            throw new IllegalStateException("Order cancelled");
        }
        
        @Override
        public boolean canCancel() {
            return false;
        }
    };
    
    public abstract OrderState next();
    public abstract boolean canCancel();
    
    public OrderState cancel() {
        if (!canCancel()) {
            throw new IllegalStateException(
                "Cannot cancel order in " + this + " state");
        }
        return CANCELLED;
    }
}

// Order class using state machine
public class Order {
    private String orderId;
    private OrderState state;
    
    public Order(String orderId) {
        this.orderId = orderId;
        this.state = OrderState.CREATED;
    }
    
    public void proceed() {
        state = state.next();
        System.out.println("Order " + orderId + " moved to " + state);
    }
    
    public void cancel() {
        state = state.cancel();
        System.out.println("Order " + orderId + " cancelled");
    }
    
    public OrderState getState() {
        return state;
    }
}

// Usage
Order order = new Order("ORD-123");
order.proceed();  // CREATED -> PENDING_PAYMENT
order.proceed();  // PENDING_PAYMENT -> PAYMENT_CONFIRMED
order.proceed();  // PAYMENT_CONFIRMED -> PROCESSING
// order.cancel();  // IllegalStateException - cannot cancel after processing
order.proceed();  // PROCESSING -> SHIPPED
order.proceed();  // SHIPPED -> DELIVERED
```

---

## Strategy Pattern with Enums

### Discount Strategy

```java
public enum DiscountStrategy {
    NONE {
        @Override
        public double apply(double price) {
            return price;
        }
    },
    
    PERCENTAGE_10 {
        @Override
        public double apply(double price) {
            return price * 0.90;
        }
    },
    
    PERCENTAGE_20 {
        @Override
        public double apply(double price) {
            return price * 0.80;
        }
    },
    
    FIXED_AMOUNT_5 {
        @Override
        public double apply(double price) {
            return Math.max(0, price - 5.0);
        }
    },
    
    BULK_DISCOUNT {
        @Override
        public double apply(double price) {
            if (price >= 100) {
                return price * 0.85;  // 15% off for bulk
            }
            return price;
        }
    },
    
    BLACK_FRIDAY {
        @Override
        public double apply(double price) {
            return price * 0.50;  // 50% off
        }
    };
    
    public abstract double apply(double price);
    
    public String formatDiscount(double originalPrice) {
        double discounted = apply(originalPrice);
        double saved = originalPrice - discounted;
        return String.format("Original: $%.2f, Final: $%.2f, Saved: $%.2f",
            originalPrice, discounted, saved);
    }
}

// Usage
double price = 100.0;

for (DiscountStrategy strategy : DiscountStrategy.values()) {
    System.out.println(strategy + ": " + strategy.formatDiscount(price));
}

// Apply specific strategy
DiscountStrategy userDiscount = DiscountStrategy.PERCENTAGE_20;
double finalPrice = userDiscount.apply(price);
```

---

## Command Pattern with Enums

### Text Editor Commands

```java
public enum EditorCommand {
    BOLD {
        @Override
        public String execute(String text) {
            return "<b>" + text + "</b>";
        }
    },
    
    ITALIC {
        @Override
        public String execute(String text) {
            return "<i>" + text + "</i>";
        }
    },
    
    UNDERLINE {
        @Override
        public String execute(String text) {
            return "<u>" + text + "</u>";
        }
    },
    
    UPPERCASE {
        @Override
        public String execute(String text) {
            return text.toUpperCase();
        }
    },
    
    LOWERCASE {
        @Override
        public String execute(String text) {
            return text.toLowerCase();
        }
    },
    
    REVERSE {
        @Override
        public String execute(String text) {
            return new StringBuilder(text).reverse().toString();
        }
    };
    
    public abstract String execute(String text);
    
    // Chain commands
    public String executeAll(String text, EditorCommand... commands) {
        String result = text;
        for (EditorCommand command : commands) {
            result = command.execute(result);
        }
        return result;
    }
}

// Usage
String text = "Hello World";

String bold = EditorCommand.BOLD.execute(text);
String italic = EditorCommand.ITALIC.execute(text);

// Chain multiple commands
String formatted = EditorCommand.BOLD.executeAll(text,
    EditorCommand.UPPERCASE,
    EditorCommand.BOLD,
    EditorCommand.ITALIC
);
```

---

## Enum for Configuration

### Environment Configuration

```java
public enum Environment {
    DEVELOPMENT("dev", "localhost", 8080, false) {
        @Override
        public String getDatabaseUrl() {
            return "jdbc:postgresql://localhost:5432/myapp_dev";
        }
    },
    
    STAGING("staging", "staging.example.com", 8080, true) {
        @Override
        public String getDatabaseUrl() {
            return "jdbc:postgresql://staging-db.example.com:5432/myapp_staging";
        }
    },
    
    PRODUCTION("prod", "example.com", 443, true) {
        @Override
        public String getDatabaseUrl() {
            return "jdbc:postgresql://prod-db.example.com:5432/myapp_prod";
        }
    };
    
    private final String name;
    private final String domain;
    private final int port;
    private final boolean sslEnabled;
    
    Environment(String name, String domain, int port, boolean sslEnabled) {
        this.name = name;
        this.domain = domain;
        this.port = port;
        this.sslEnabled = sslEnabled;
    }
    
    public abstract String getDatabaseUrl();
    
    public String getName() { return name; }
    public String getDomain() { return domain; }
    public int getPort() { return port; }
    public boolean isSslEnabled() { return sslEnabled; }
    
    public String getBaseUrl() {
        String protocol = sslEnabled ? "https" : "http";
        return String.format("%s://%s:%d", protocol, domain, port);
    }
    
    public boolean isProduction() {
        return this == PRODUCTION;
    }
}

// Usage
Environment env = Environment.DEVELOPMENT;

String dbUrl = env.getDatabaseUrl();
String baseUrl = env.getBaseUrl();

if (env.isProduction()) {
    // Production-specific logic
}
```

---

## Enum Best Practices

### ✅ DO

1. **Use enums for fixed sets of constants**
   ```java
   public enum Status { ACTIVE, INACTIVE, SUSPENDED }
   ```

2. **Add behavior to enums**
   ```java
   public enum Operation {
       PLUS { double apply(double x, double y) { return x + y; } },
       MINUS { double apply(double x, double y) { return x - y; } };
       abstract double apply(double x, double y);
   }
   ```

3. **Use EnumSet and EnumMap for performance**
   ```java
   EnumSet<Day> weekend = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
   EnumMap<Status, Integer> counts = new EnumMap<>(Status.class);
   ```

4. **Use enum singleton pattern**
   ```java
   public enum Singleton { INSTANCE; }
   ```

5. **Implement interfaces for polymorphism**
   ```java
   public enum Priority implements Comparable<Priority> { }
   ```

6. **Use switch expressions (Java 14+)**
   ```java
   String msg = switch (status) {
       case PENDING -> "Waiting";
       case DONE -> "Complete";
   };
   ```

7. **Document enum constants**
   ```java
   /**
    * Represents the status of an order.
    */
   public enum OrderStatus {
       /** Order has been created but not paid */
       PENDING,
       /** Order is being processed */
       PROCESSING
   }
   ```

8. **Use descriptive names**
   ```java
   public enum HttpMethod { GET, POST, PUT, DELETE }
   ```

9. **Provide reverse lookup when needed**
   ```java
   public static Status fromCode(int code) {
       for (Status s : values()) {
           if (s.code == code) return s;
       }
       throw new IllegalArgumentException();
   }
   ```

10. **Use @Override for clarity**
    ```java
    @Override
    public String toString() { return name().toLowerCase(); }
    ```

### ❌ DON'T

1. **Don't use ordinal() for logic**
   ```java
   // ❌ BAD
   if (status.ordinal() > 3) { }
   
   // ✅ GOOD
   if (status.isComplete()) { }
   ```

2. **Don't add mutable fields**
   ```java
   // ❌ BAD
   public enum Status {
       ACTIVE;
       private int count;  // Mutable!
       public void incrementCount() { count++; }
   }
   
   // ✅ GOOD - use immutable fields
   public enum Status {
       ACTIVE(10);
       private final int maxRetries;
   }
   ```

3. **Don't use valueOf() without validation**
   ```java
   // ❌ BAD
   Status status = Status.valueOf(userInput);  // May throw exception
   
   // ✅ GOOD
   try {
       Status status = Status.valueOf(userInput);
   } catch (IllegalArgumentException e) {
       // Handle invalid input
   }
   ```

4. **Don't serialize enums manually**
   ```java
   // Enums are serializable by default
   // Don't implement writeObject/readObject
   ```

5. **Don't use == null check**
   ```java
   // ❌ Unnecessary
   if (status != null && status == Status.ACTIVE) { }
   
   // ✅ Enum can never be null in well-designed code
   if (status == Status.ACTIVE) { }
   ```

---

## Common Pitfalls

### Pitfall 1: Ordinal Dependency

```java
// ❌ BAD: Depending on ordinal
public enum Priority {
    LOW, MEDIUM, HIGH, CRITICAL
}

if (priority.ordinal() >= 2) {  // Fragile!
    // High priority logic
}

// ✅ GOOD: Explicit logic
public enum Priority {
    LOW(1), MEDIUM(5), HIGH(10), CRITICAL(20);
    
    private final int level;
    Priority(int level) { this.level = level; }
    
    public boolean isHighPriority() {
        return level >= 10;
    }
}
```

### Pitfall 2: Missing null Check

```java
// ❌ BAD: No null handling
public void process(Status status) {
    switch (status) {  // NullPointerException if status is null!
        case ACTIVE -> handleActive();
        case INACTIVE -> handleInactive();
    }
}

// ✅ GOOD: Handle null
public void process(Status status) {
    if (status == null) {
        throw new IllegalArgumentException("Status cannot be null");
    }
    // Or use Objects.requireNonNull(status)
    
    switch (status) {
        case ACTIVE -> handleActive();
        case INACTIVE -> handleInactive();
    }
}
```

### Pitfall 3: valueOf() without Error Handling

```java
// ❌ BAD: No error handling
Status status = Status.valueOf(input);  // Throws IllegalArgumentException

// ✅ GOOD: Safe parsing
public static Optional<Status> parse(String input) {
    try {
        return Optional.of(Status.valueOf(input));
    } catch (IllegalArgumentException e) {
        return Optional.empty();
    }
}

// Usage
Optional<Status> status = Status.parse(userInput);
status.ifPresent(this::processStatus);
```

---

## Conclusion

**Key Takeaways:**

1. **Type Safety**: Enums provide compile-time type safety
2. **Behavior**: Enums can have fields, methods, and even abstract methods
3. **Patterns**: Perfect for singleton, strategy, state machine, command patterns
4. **Performance**: EnumSet and EnumMap are highly optimized
5. **Immutability**: Always use immutable fields in enums
6. **Avoid ordinal()**: Don't depend on ordinal values
7. **State Machines**: Excellent for modeling state transitions
8. **Configuration**: Great for environment-specific configs

**Remember**: Enums are full-fledged classes in Java. Use them not just as constants, but as a powerful tool for implementing type-safe, behavior-rich abstractions!
