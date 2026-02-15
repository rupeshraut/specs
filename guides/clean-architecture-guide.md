# Clean & Hexagonal Architecture Deep Dive

A comprehensive guide to implementing Clean Architecture and Hexagonal Architecture in enterprise Java applications with practical patterns and real-world examples.

---

## Table of Contents

1. [Architecture Fundamentals](#architecture-fundamentals)
2. [The Dependency Rule](#the-dependency-rule)
3. [Hexagonal Architecture](#hexagonal-architecture)
4. [Layer Organization](#layer-organization)
5. [Domain Layer](#domain-layer)
6. [Application Layer](#application-layer)
7. [Ports & Adapters](#ports--adapters)
8. [Infrastructure Layer](#infrastructure-layer)
9. [Package Structure](#package-structure)
10. [Dependency Injection](#dependency-injection)
11. [Testing Strategy](#testing-strategy)
12. [Migration Patterns](#migration-patterns)
13. [Common Pitfalls](#common-pitfalls)

---

## Architecture Fundamentals

### Why Clean/Hexagonal Architecture?

```
Traditional Layered Architecture Problems:
❌ Domain logic leaks into controllers
❌ Business rules coupled to frameworks
❌ Hard to test without infrastructure
❌ Difficult to change databases or frameworks
❌ Technology drives design

Clean/Hexagonal Architecture Benefits:
✅ Domain logic isolated and pure
✅ Framework-independent business rules
✅ Easy to test without infrastructure
✅ Swap databases/frameworks easily
✅ Business drives design
```

### Core Principles

```
1. Independence
   - Independent of Frameworks
   - Independent of UI
   - Independent of Database
   - Independent of External Services
   - Testable in isolation

2. The Dependency Rule
   Source code dependencies point INWARD
   Inner layers know nothing about outer layers

3. Screaming Architecture
   Architecture reveals what the system does
   Not what frameworks it uses
```

---

## The Dependency Rule

### Dependency Flow

```
┌─────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE                         │
│  (Frameworks, Drivers, Web, DB, External APIs)         │
│                                                         │
│  ┌───────────────────────────────────────────────┐    │
│  │            INTERFACE ADAPTERS                  │    │
│  │   (Controllers, Presenters, Gateways)         │    │
│  │                                                │    │
│  │  ┌──────────────────────────────────────┐    │    │
│  │  │      APPLICATION (Use Cases)         │    │    │
│  │  │                                       │    │    │
│  │  │  ┌────────────────────────────┐     │    │    │
│  │  │  │   DOMAIN (Entities)        │     │    │    │
│  │  │  │   - Business Rules         │     │    │    │
│  │  │  │   - Enterprise Logic       │     │    │    │
│  │  │  └────────────────────────────┘     │    │    │
│  │  │                                       │    │    │
│  │  └──────────────────────────────────────┘    │    │
│  │                                                │    │
│  └───────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘

Dependencies point INWARD →
```

### Code Example

```java
// ✅ CORRECT: Inner layer doesn't know about outer layers
// Domain Layer
public class Order {
    private String id;
    private List<OrderItem> items;
    private OrderStatus status;

    // Pure business logic - no framework dependencies
    public void confirm() {
        if (items.isEmpty()) {
            throw new OrderCannotBeEmptyException();
        }
        if (status != OrderStatus.PENDING) {
            throw new OrderAlreadyProcessedException();
        }
        this.status = OrderStatus.CONFIRMED;
    }
}

// Application Layer - depends on Domain
public class ConfirmOrderUseCase {
    private final OrderRepository repository; // Port (interface)

    public void execute(String orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.confirm();

        repository.save(order);
    }
}

// Infrastructure Layer - depends on Application
public class MongoOrderRepository implements OrderRepository {
    private final MongoTemplate mongoTemplate;

    @Override
    public Optional<Order> findById(String id) {
        OrderDocument doc = mongoTemplate.findById(id, OrderDocument.class);
        return Optional.ofNullable(doc).map(this::toDomain);
    }
}
```

---

## Hexagonal Architecture

### Ports and Adapters

```
                    ┌─────────────────────┐
                    │   REST Controller   │
                    │    (Adapter)        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Application       │
                    │   (Use Cases)       │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
        ┌───────────┤      DOMAIN         ├───────────┐
        │           │  (Business Logic)   │           │
        │           └─────────────────────┘           │
        │                                             │
┌───────▼────────┐                          ┌────────▼────────┐
│ OrderRepository│                          │ NotificationPort│
│    (Port)      │                          │    (Port)       │
└───────┬────────┘                          └────────┬────────┘
        │                                            │
┌───────▼────────┐                          ┌────────▼────────┐
│  MongoDB Repo  │                          │  Email Service  │
│   (Adapter)    │                          │   (Adapter)     │
└────────────────┘                          └─────────────────┘

Ports = Interfaces (what the application needs)
Adapters = Implementations (how it's done)
```

### Primary vs Secondary Ports

```java
// PRIMARY PORT (Driving) - Use Case interface
public interface CreateOrderUseCase {
    OrderResponse execute(CreateOrderRequest request);
}

// SECONDARY PORT (Driven) - Repository interface
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(String id);
}

// PRIMARY ADAPTER - REST Controller
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrder;

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> create(@RequestBody CreateOrderRequest request) {
        OrderResponse response = createOrder.execute(request);
        return ResponseEntity.ok(response);
    }
}

// SECONDARY ADAPTER - MongoDB Repository
@Component
public class MongoOrderRepository implements OrderRepository {
    private final MongoTemplate mongoTemplate;

    @Override
    public Order save(Order order) {
        OrderDocument doc = toDocument(order);
        mongoTemplate.save(doc);
        return order;
    }
}
```

---

## Domain Layer

### Entities

```java
// Rich domain model - behavior, not just data
public class Payment {
    private PaymentId id;
    private Money amount;
    private PaymentStatus status;
    private Instant createdAt;
    private List<PaymentEvent> events = new ArrayList<>();

    // Factory method
    public static Payment create(Money amount, String userId) {
        Payment payment = new Payment();
        payment.id = PaymentId.generate();
        payment.amount = amount;
        payment.status = PaymentStatus.PENDING;
        payment.createdAt = Instant.now();

        payment.addEvent(new PaymentCreatedEvent(payment.id, amount));

        return payment;
    }

    // Business logic
    public void process() {
        if (status != PaymentStatus.PENDING) {
            throw new PaymentAlreadyProcessedException(id);
        }

        if (amount.isNegative()) {
            throw new InvalidPaymentAmountException();
        }

        this.status = PaymentStatus.PROCESSING;
        addEvent(new PaymentProcessingStartedEvent(id));
    }

    public void complete() {
        if (status != PaymentStatus.PROCESSING) {
            throw new IllegalStateException("Payment not in processing state");
        }

        this.status = PaymentStatus.COMPLETED;
        addEvent(new PaymentCompletedEvent(id, Instant.now()));
    }

    // Invariants protected
    private void addEvent(PaymentEvent event) {
        events.add(event);
    }

    // Getters only - no setters that bypass business rules
    public PaymentId getId() { return id; }
    public PaymentStatus getStatus() { return status; }
    public List<PaymentEvent> getEvents() { return List.copyOf(events); }
}
```

### Value Objects

```java
// Immutable value object
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");

        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("Invalid amount scale");
        }
    }

    public static Money of(BigDecimal amount, String currencyCode) {
        return new Money(amount, Currency.getInstance(currencyCode));
    }

    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public boolean isNegative() {
        return amount.compareTo(BigDecimal.ZERO) < 0;
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException();
        }
    }
}
```

---

## Application Layer

### Use Cases

```java
// Use case with clear input/output
public class ProcessPaymentUseCase {

    private final PaymentRepository paymentRepository;
    private final PaymentGateway paymentGateway;
    private final EventPublisher eventPublisher;

    public ProcessPaymentResponse execute(ProcessPaymentRequest request) {
        // 1. Load aggregate
        Payment payment = paymentRepository.findById(request.paymentId())
            .orElseThrow(() -> new PaymentNotFoundException(request.paymentId()));

        // 2. Execute business logic
        payment.process();

        // 3. Call external service
        PaymentGatewayResponse gatewayResponse = paymentGateway.charge(
            payment.getAmount(),
            request.paymentMethod()
        );

        if (gatewayResponse.isSuccessful()) {
            payment.complete();
        } else {
            payment.fail(gatewayResponse.getReason());
        }

        // 4. Save changes
        paymentRepository.save(payment);

        // 5. Publish events
        payment.getEvents().forEach(eventPublisher::publish);

        // 6. Return response
        return new ProcessPaymentResponse(
            payment.getId(),
            payment.getStatus(),
            gatewayResponse.getTransactionId()
        );
    }
}
```

### DTOs

```java
// Input DTO
public record ProcessPaymentRequest(
    String paymentId,
    PaymentMethod paymentMethod
) {
    public ProcessPaymentRequest {
        Objects.requireNonNull(paymentId);
        Objects.requireNonNull(paymentMethod);
    }
}

// Output DTO
public record ProcessPaymentResponse(
    String paymentId,
    PaymentStatus status,
    String transactionId
) {}
```

---

## Package Structure

### Feature-Based (Recommended)

```
src/main/java/com/company/payment/
├── domain/
│   ├── payment/
│   │   ├── Payment.java (Entity)
│   │   ├── PaymentId.java (Value Object)
│   │   ├── PaymentStatus.java (Enum)
│   │   ├── Money.java (Value Object)
│   │   └── PaymentRepository.java (Port)
│   └── order/
│       ├── Order.java
│       └── OrderRepository.java
│
├── application/
│   ├── payment/
│   │   ├── ProcessPaymentUseCase.java
│   │   ├── ProcessPaymentRequest.java
│   │   ├── ProcessPaymentResponse.java
│   │   └── PaymentGateway.java (Port)
│   └── order/
│       └── CreateOrderUseCase.java
│
├── infrastructure/
│   ├── web/
│   │   └── payment/
│   │       └── PaymentController.java (Primary Adapter)
│   │
│   ├── persistence/
│   │   ├── payment/
│   │   │   ├── MongoPaymentRepository.java (Secondary Adapter)
│   │   │   ├── PaymentDocument.java (DB Model)
│   │   │   └── PaymentMapper.java
│   │   └── order/
│   │       └── MongoOrderRepository.java
│   │
│   ├── messaging/
│   │   └── kafka/
│   │       └── KafkaEventPublisher.java
│   │
│   └── external/
│       └── stripe/
│           └── StripePaymentGateway.java
│
└── config/
    ├── ApplicationConfig.java
    └── InfrastructureConfig.java
```

---

## Dependency Injection

### Spring Configuration

```java
@Configuration
public class PaymentConfig {

    // Use Cases
    @Bean
    public ProcessPaymentUseCase processPaymentUseCase(
            PaymentRepository paymentRepository,
            PaymentGateway paymentGateway,
            EventPublisher eventPublisher) {

        return new ProcessPaymentUseCase(
            paymentRepository,
            paymentGateway,
            eventPublisher
        );
    }

    // Repositories (Adapters)
    @Bean
    public PaymentRepository paymentRepository(MongoTemplate mongoTemplate) {
        return new MongoPaymentRepository(mongoTemplate);
    }

    // External Services (Adapters)
    @Bean
    public PaymentGateway paymentGateway(StripeClient stripeClient) {
        return new StripePaymentGateway(stripeClient);
    }
}
```

---

## Testing Strategy

### Domain Tests (Pure Unit Tests)

```java
class PaymentTest {

    @Test
    void shouldCreatePendingPayment() {
        Money amount = Money.of(new BigDecimal("100.00"), "USD");

        Payment payment = Payment.create(amount, "user123");

        assertThat(payment.getStatus()).isEqualTo(PaymentStatus.PENDING);
        assertThat(payment.getAmount()).isEqualTo(amount);
        assertThat(payment.getEvents()).hasSize(1);
    }

    @Test
    void shouldProcessPendingPayment() {
        Payment payment = Payment.create(
            Money.of(new BigDecimal("100.00"), "USD"),
            "user123"
        );

        payment.process();

        assertThat(payment.getStatus()).isEqualTo(PaymentStatus.PROCESSING);
    }

    @Test
    void shouldThrowExceptionWhenProcessingNonPendingPayment() {
        Payment payment = Payment.create(
            Money.of(new BigDecimal("100.00"), "USD"),
            "user123"
        );
        payment.process();
        payment.complete();

        assertThatThrownBy(() -> payment.process())
            .isInstanceOf(PaymentAlreadyProcessedException.class);
    }
}
```

### Use Case Tests (Integration Tests)

```java
class ProcessPaymentUseCaseTest {

    private PaymentRepository paymentRepository;
    private PaymentGateway paymentGateway;
    private EventPublisher eventPublisher;
    private ProcessPaymentUseCase useCase;

    @BeforeEach
    void setUp() {
        paymentRepository = mock(PaymentRepository.class);
        paymentGateway = mock(PaymentGateway.class);
        eventPublisher = mock(EventPublisher.class);

        useCase = new ProcessPaymentUseCase(
            paymentRepository,
            paymentGateway,
            eventPublisher
        );
    }

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Given
        Payment payment = Payment.create(
            Money.of(new BigDecimal("100.00"), "USD"),
            "user123"
        );
        when(paymentRepository.findById(payment.getId()))
            .thenReturn(Optional.of(payment));

        when(paymentGateway.charge(any(), any()))
            .thenReturn(PaymentGatewayResponse.successful("txn123"));

        // When
        ProcessPaymentResponse response = useCase.execute(
            new ProcessPaymentRequest(payment.getId(), PaymentMethod.CARD)
        );

        // Then
        assertThat(response.status()).isEqualTo(PaymentStatus.COMPLETED);
        verify(paymentRepository).save(any(Payment.class));
        verify(eventPublisher, atLeastOnce()).publish(any());
    }
}
```

---

## Migration Patterns

### Strangler Fig Pattern

```java
// Step 1: Create new architecture alongside old
@RestController
public class PaymentController {

    private final LegacyPaymentService legacyService;
    private final ProcessPaymentUseCase newUseCase;
    private final FeatureToggle featureToggle;

    @PostMapping("/payments/process")
    public ResponseEntity<?> processPayment(@RequestBody PaymentRequest request) {

        if (featureToggle.isEnabled("new-payment-architecture")) {
            // New clean architecture
            ProcessPaymentResponse response = newUseCase.execute(
                toUseCaseRequest(request)
            );
            return ResponseEntity.ok(response);
        } else {
            // Old legacy code
            LegacyPaymentResponse response = legacyService.process(request);
            return ResponseEntity.ok(response);
        }
    }
}

// Step 2: Run in parallel mode (shadow traffic)
if (featureToggle.isEnabled("shadow-new-architecture")) {
    CompletableFuture.runAsync(() -> {
        try {
            newUseCase.execute(toUseCaseRequest(request));
        } catch (Exception e) {
            log.warn("New architecture failed (shadow mode)", e);
        }
    });
}

// Step 3: Switch traffic
// Step 4: Remove legacy code
```

---

## Best Practices

### ✅ DO

- Keep domain layer pure (no framework dependencies)
- Use value objects for domain concepts
- Make entities responsible for invariants
- Define clear ports (interfaces)
- Test business logic without infrastructure
- Use factories for complex object creation
- Validate at boundaries
- Return immutable collections from entities
- Use package-private for implementation details

### ❌ DON'T

- Put business logic in controllers
- Let infrastructure leak into domain
- Use JPA annotations on domain entities
- Expose setters that bypass business rules
- Test through frameworks
- Use anemic domain models
- Skip input validation
- Return mutable state from entities
- Make everything public

---

*This guide provides comprehensive patterns for implementing Clean and Hexagonal Architecture in enterprise Java applications. Start small, prove the value, then expand adoption.*
