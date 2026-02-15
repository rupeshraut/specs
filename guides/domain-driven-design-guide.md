# Domain-Driven Design (DDD) Complete Guide

A comprehensive guide to implementing Domain-Driven Design in enterprise Java applications with tactical and strategic patterns.

---

## Table of Contents

1. [DDD Fundamentals](#ddd-fundamentals)
2. [Strategic Design](#strategic-design)
3. [Bounded Contexts](#bounded-contexts)
4. [Context Mapping](#context-mapping)
5. [Tactical Design](#tactical-design)
6. [Entities](#entities)
7. [Value Objects](#value-objects)
8. [Aggregates](#aggregates)
9. [Domain Events](#domain-events)
10. [Repositories](#repositories)
11. [Domain Services](#domain-services)
12. [Application Services](#application-services)
13. [Factories](#factories)
14. [DDD with Spring Boot](#ddd-with-spring-boot)

---

## DDD Fundamentals

### What is Domain-Driven Design?

```
DDD is a software development approach that:

1. Focuses on Core Domain
   - Understand business deeply
   - Model complex domains
   - Align code with business

2. Collaboration
   - Developers + Domain Experts
   - Ubiquitous Language
   - Shared understanding

3. Strategic Design
   - Bounded Contexts
   - Context Maps
   - Domain separation

4. Tactical Patterns
   - Entities, Value Objects
   - Aggregates
   - Domain Events
```

### When to Use DDD

```
✅ Use DDD when:
- Complex business domain
- Long-lived project
- Evolving requirements
- Need business/tech alignment
- Multiple domain experts

❌ Don't use DDD when:
- Simple CRUD application
- Short-lived project
- Clear, stable requirements
- Technical focus (no domain)
- Solo developer
```

---

## Strategic Design

### Ubiquitous Language

```java
// ❌ BAD: Technical language
public class DataProcessor {
    public void process(Map<String, Object> data) {
        // What does this do in business terms?
    }
}

// ✅ GOOD: Business language
public class PaymentProcessor {
    public void authorizePayment(Payment payment) {
        // Clear business intent
    }
}

// Ubiquitous Language in action:
// "When a customer places an order, we need to authorize
//  the payment and reserve inventory before confirming
//  the order."

public class Order {
    public void confirm() {
        if (!this.isPaymentAuthorized()) {
            throw new PaymentNotAuthorizedException();
        }

        if (!this.isInventoryReserved()) {
            throw new InventoryNotReservedException();
        }

        this.status = OrderStatus.CONFIRMED;
        this.addDomainEvent(new OrderConfirmedEvent(this.id));
    }
}
```

---

## Bounded Contexts

### Identifying Boundaries

```
E-commerce Example:

┌─────────────────────────────────────────────────────────┐
│                    E-Commerce System                     │
│                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────┐│
│  │   Shopping     │  │   Inventory    │  │  Shipping  ││
│  │   Context      │  │   Context      │  │  Context   ││
│  │                │  │                │  │            ││
│  │ - Cart         │  │ - Stock        │  │ - Shipment ││
│  │ - Product      │  │ - Warehouse    │  │ - Tracking ││
│  │ - Catalog      │  │ - SKU          │  │ - Carrier  ││
│  └────────────────┘  └────────────────┘  └────────────┘│
│         │                    │                  │        │
│         └────────────────────┴──────────────────┘        │
│                             │                            │
│                  ┌──────────▼──────────┐                │
│                  │   Payment Context   │                │
│                  │                     │                │
│                  │ - Payment           │                │
│                  │ - Transaction       │                │
│                  │ - Refund            │                │
│                  └─────────────────────┘                │
└─────────────────────────────────────────────────────────┘

"Product" means different things in each context:
- Shopping: Display info, price, images
- Inventory: Stock levels, warehouse location
- Shipping: Weight, dimensions, packaging
```

### Bounded Context Implementation

```java
// Shopping Context
package com.company.shopping;

public class Product {
    private ProductId id;
    private String name;
    private Money price;
    private String description;
    private List<String> imageUrls;

    // Shopping-specific behavior
    public boolean isAvailableForPurchase() {
        return price.isPositive();
    }
}

// Inventory Context
package com.company.inventory;

public class Product {
    private SKU sku;
    private int stockLevel;
    private WarehouseLocation location;
    private int reorderPoint;

    // Inventory-specific behavior
    public boolean needsReordering() {
        return stockLevel <= reorderPoint;
    }
}

// Different models, same concept, different contexts
```

---

## Context Mapping

### Relationship Patterns

```
1. Partnership
   Both contexts succeed/fail together

2. Shared Kernel
   Shared subset of domain model

3. Customer-Supplier
   Downstream depends on upstream

4. Conformist
   Downstream conforms to upstream

5. Anti-Corruption Layer (ACL)
   Translation layer to protect domain

6. Separate Ways
   No integration, independent contexts
```

### Anti-Corruption Layer

```java
// External Payment Gateway (out of our control)
public interface ExternalPaymentGateway {
    GatewayResponse charge(GatewayRequest request);
}

// Anti-Corruption Layer
@Component
public class PaymentGatewayAdapter implements PaymentGateway {

    private final ExternalPaymentGateway externalGateway;

    @Override
    public PaymentResult processPayment(Payment payment) {
        // Translate from our domain to external format
        GatewayRequest request = toGatewayRequest(payment);

        // Call external service
        GatewayResponse response = externalGateway.charge(request);

        // Translate back to our domain
        return toPaymentResult(response);
    }

    private GatewayRequest toGatewayRequest(Payment payment) {
        return GatewayRequest.builder()
            .amount(payment.getAmount().getValue())
            .currency(payment.getAmount().getCurrency().getCurrencyCode())
            .cardNumber(payment.getPaymentMethod().getCardNumber())
            .build();
    }

    private PaymentResult toPaymentResult(GatewayResponse response) {
        if (response.isSuccessful()) {
            return PaymentResult.success(
                response.getTransactionId(),
                Instant.parse(response.getTimestamp())
            );
        } else {
            return PaymentResult.failure(
                PaymentFailureReason.valueOf(response.getErrorCode())
            );
        }
    }
}
```

---

## Entities

### Identity

```java
public abstract class Entity<ID> {
    protected ID id;

    protected Entity(ID id) {
        this.id = Objects.requireNonNull(id, "ID cannot be null");
    }

    public ID getId() {
        return id;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;

        Entity<?> other = (Entity<?>) obj;
        return id.equals(other.id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}

// Payment Entity
public class Payment extends Entity<PaymentId> {

    private Money amount;
    private PaymentStatus status;
    private Instant createdAt;

    private Payment(PaymentId id) {
        super(id);
    }

    public static Payment create(Money amount) {
        Payment payment = new Payment(PaymentId.generate());
        payment.amount = amount;
        payment.status = PaymentStatus.PENDING;
        payment.createdAt = Instant.now();
        return payment;
    }

    // Business behavior
    public void authorize() {
        if (status != PaymentStatus.PENDING) {
            throw new PaymentAlreadyAuthorizedException(getId());
        }

        this.status = PaymentStatus.AUTHORIZED;
    }
}
```

---

## Value Objects

### Immutable Values

```java
// Money Value Object
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");

        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException("Invalid amount scale");
        }
    }

    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public static Money of(String amount, String currencyCode) {
        return new Money(
            new BigDecimal(amount),
            Currency.getInstance(currencyCode)
        );
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(
            this.amount.add(other.amount),
            this.currency
        );
    }

    public Money multiply(BigDecimal multiplier) {
        return new Money(
            this.amount.multiply(multiplier),
            this.currency
        );
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException(
                "Cannot operate on different currencies: " +
                currency + " and " + other.currency
            );
        }
    }
}

// Usage
Money price = Money.of("99.99", "USD");
Money tax = price.multiply(new BigDecimal("0.10"));
Money total = price.add(tax);
```

### Email Value Object

```java
public record Email(String value) {

    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"
    );

    public Email {
        Objects.requireNonNull(value, "Email cannot be null");

        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException(value);
        }
    }

    public static Email of(String email) {
        return new Email(email.toLowerCase().trim());
    }

    public String getDomain() {
        return value.substring(value.indexOf('@') + 1);
    }
}
```

---

## Aggregates

### Aggregate Root

```java
// Order Aggregate
public class Order extends Entity<OrderId> {

    private UserId userId;
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;
    private Money totalAmount;
    private Address shippingAddress;

    // Aggregate invariant: Cannot add items after confirmation
    public void addItem(ProductId productId, int quantity, Money price) {
        if (status != OrderStatus.DRAFT) {
            throw new OrderAlreadyConfirmedException(getId());
        }

        if (quantity <= 0) {
            throw new InvalidQuantityException(quantity);
        }

        OrderItem item = new OrderItem(productId, quantity, price);
        items.add(item);

        // Recalculate total (invariant)
        recalculateTotal();
    }

    public void confirm() {
        if (items.isEmpty()) {
            throw new EmptyOrderException(getId());
        }

        if (shippingAddress == null) {
            throw new MissingShippingAddressException(getId());
        }

        this.status = OrderStatus.CONFIRMED;

        // Domain event
        this.addDomainEvent(new OrderConfirmedEvent(
            getId(),
            userId,
            totalAmount
        ));
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getLineTotal)
            .reduce(Money.zero(getCurrency()), Money::add);
    }

    // Only access items through aggregate root
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}

// OrderItem is part of Order aggregate (not a separate aggregate)
class OrderItem {
    private ProductId productId;
    private int quantity;
    private Money unitPrice;

    public Money getLineTotal() {
        return unitPrice.multiply(new BigDecimal(quantity));
    }
}
```

### Aggregate Design Rules

```
1. Reference other aggregates by ID only
   ✅ Order has UserId (reference by ID)
   ❌ Order has User object

2. One transaction = One aggregate
   ✅ Save entire Order aggregate in one transaction
   ❌ Update Order and Inventory in same transaction

3. Small aggregates preferred
   ✅ Order with OrderItems (closely related)
   ❌ Order with User, Products, Inventory, etc.

4. Protect invariants
   ✅ Order ensures items exist before confirmation
   ✅ Payment ensures amount > 0

5. Aggregate root controls access
   ✅ Add items through Order.addItem()
   ❌ Direct access to items list
```

---

## Domain Events

### Event Implementation

```java
// Domain Event
public record OrderConfirmedEvent(
    String eventId,
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    Instant occurredAt
) {
    public OrderConfirmedEvent(OrderId orderId, UserId userId, Money totalAmount) {
        this(
            UUID.randomUUID().toString(),
            orderId,
            userId,
            totalAmount,
            Instant.now()
        );
    }
}

// Entity with Events
public abstract class AggregateRoot<ID> extends Entity<ID> {

    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected AggregateRoot(ID id) {
        super(id);
    }

    protected void addDomainEvent(DomainEvent event) {
        domainEvents.add(event);
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        domainEvents.clear();
    }
}

// Publishing Events
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void confirmOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.confirm();

        orderRepository.save(order);

        // Publish domain events
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        order.clearDomainEvents();
    }
}
```

---

## Repositories

### Repository Interface

```java
// Domain layer - Port
public interface OrderRepository {

    Order save(Order order);

    Optional<Order> findById(OrderId id);

    List<Order> findByUserId(UserId userId);

    void delete(OrderId id);
}

// Infrastructure layer - Adapter
@Repository
public class MongoOrderRepository implements OrderRepository {

    private final MongoTemplate mongoTemplate;
    private final OrderMapper mapper;

    @Override
    public Order save(Order order) {
        OrderDocument doc = mapper.toDocument(order);
        mongoTemplate.save(doc);
        return order;
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        OrderDocument doc = mongoTemplate.findById(
            id.getValue(),
            OrderDocument.class
        );

        return Optional.ofNullable(doc)
            .map(mapper::toDomain);
    }
}
```

---

## Domain Services

### When to Use Domain Services

```java
// ✅ Use Domain Service when:
// - Operation spans multiple aggregates
// - Doesn't naturally belong to one entity
// - Stateless operation with domain logic

@DomainService
public class PaymentAuthorizationService {

    // Coordinates between Payment and PaymentGateway
    public PaymentAuthorization authorizePayment(
            Payment payment,
            PaymentGateway gateway) {

        // Validate payment
        if (!payment.canBeAuthorized()) {
            throw new PaymentNotAuthorizableException();
        }

        // Call gateway
        GatewayResponse response = gateway.authorize(
            payment.getAmount(),
            payment.getPaymentMethod()
        );

        // Create authorization result
        if (response.isSuccessful()) {
            return PaymentAuthorization.approved(
                payment.getId(),
                response.getAuthorizationCode()
            );
        } else {
            return PaymentAuthorization.declined(
                payment.getId(),
                response.getDeclineReason()
            );
        }
    }
}
```

---

## Best Practices

### ✅ DO

- Use ubiquitous language everywhere
- Make implicit concepts explicit
- Protect aggregate invariants
- Keep aggregates small
- Use value objects for domain concepts
- Raise domain events for significant changes
- Model side effects as domain events
- Separate domain from infrastructure
- Test domain logic in isolation

### ❌ DON'T

- Use anemic domain model
- Put business logic in services
- Reference other aggregates directly
- Update multiple aggregates in one transaction
- Leak infrastructure into domain
- Use primitive types for domain concepts
- Make entities mutable everywhere
- Skip domain events
- Couple domain to framework

---

*This guide provides comprehensive patterns for implementing Domain-Driven Design. DDD requires discipline but pays dividends in complex domains through better modeling and maintainability.*
