# Design Patterns Decision Tree

A comprehensive guide to help you choose the right design pattern(s) for your specific scenario.

---

## Table of Contents

1. [Quick Start Decision Flow](#quick-start-decision-flow)
2. [Object Creation Decisions](#object-creation-decisions)
3. [Structural Decisions](#structural-decisions)
4. [Behavioral Decisions](#behavioral-decisions)
5. [Enterprise & Architectural Patterns](#enterprise--architectural-patterns)
6. [Combination Patterns](#combination-patterns)
7. [Anti-Pattern Warnings](#anti-pattern-warnings)

---

## Quick Start Decision Flow

```
START: What is your primary challenge?
│
├─ Creating Objects? ────────────────────► Go to [Object Creation Decisions]
│
├─ Composing Objects/Classes? ───────────► Go to [Structural Decisions]
│
├─ Object Behavior/Communication? ────────► Go to [Behavioral Decisions]
│
└─ Enterprise Architecture? ──────────────► Go to [Enterprise & Architectural Patterns]
```

---

## Object Creation Decisions

### Decision Tree

```
Need to create objects?
│
├─ Q: Is object creation complex or expensive?
│  │
│  ├─ YES: Q: Do you need different representations of the same construction?
│  │  │
│  │  ├─ YES: **BUILDER PATTERN**
│  │  │     Example: Building complex objects like SQL queries, HTTP requests
│  │  │     
│  │  └─ NO: Q: Can object creation be deferred?
│  │     │
│  │     ├─ YES: **LAZY INITIALIZATION + SINGLETON/FACTORY**
│  │     │
│  │     └─ NO: Q: Are you creating families of related objects?
│  │        │
│  │        ├─ YES: **ABSTRACT FACTORY**
│  │        │     Example: UI components for different OS (Windows/Mac)
│  │        │
│  │        └─ NO: **FACTORY METHOD**
│  │              Example: Creating different payment processors
│  │
│  └─ NO: Q: Do you need only one instance globally?
│     │
│     ├─ YES: **SINGLETON PATTERN**
│     │     ⚠️  WARNING: Consider dependency injection instead
│     │     Use cases: Logger, Configuration, Connection Pool
│     │
│     └─ NO: Q: Do you need to create objects by copying existing ones?
│        │
│        ├─ YES: **PROTOTYPE PATTERN**
│        │     Example: Cloning database configurations, game objects
│        │
│        └─ NO: Q: Do you need to separate object creation from business logic?
│           │
│           ├─ YES: **FACTORY METHOD or STATIC FACTORY**
│           │
│           └─ NO: Use direct instantiation (new Constructor())
```

### Pattern Comparison Matrix

| Scenario | Pattern | When to Use | Java Example |
|----------|---------|-------------|--------------|
| Complex object with many parameters | **Builder** | 4+ constructor params, optional parameters | `HttpRequest.newBuilder()` |
| One instance per JVM | **Singleton** | Configuration, thread pools, caches | `Runtime.getRuntime()` |
| Create related object families | **Abstract Factory** | Cross-platform UI, database drivers | JDBC `DriverManager` |
| Defer instantiation to subclasses | **Factory Method** | Framework defining creation points | `Collection.iterator()` |
| Clone expensive objects | **Prototype** | Avoid re-initialization overhead | `Object.clone()` |
| Hide complex initialization | **Static Factory** | Alternative to constructors | `List.of()`, `Optional.of()` |

---

## Structural Decisions

### Decision Tree

```
Need to compose objects/classes?
│
├─ Q: Do you need to make incompatible interfaces work together?
│  │
│  ├─ YES: **ADAPTER PATTERN**
│  │     Example: Legacy system integration, third-party library wrapping
│  │     Combine with: Factory (to create adapters)
│  │
│  └─ NO: Q: Do you need to add behavior without modifying class?
│     │
│     ├─ YES: **DECORATOR PATTERN**
│     │     Example: I/O streams, HTTP request/response interceptors
│     │     Combine with: Factory (to create decorator chains)
│     │
│     └─ NO: Q: Do you need a simplified interface to complex subsystem?
│        │
│        ├─ YES: **FACADE PATTERN**
│        │     Example: Payment processing, video conversion, email service
│        │     Combine with: Singleton (single facade instance)
│        │
│        └─ NO: Q: Do you need to compose objects into tree structures?
│           │
│           ├─ YES: **COMPOSITE PATTERN**
│           │     Example: File systems, UI component hierarchy, org charts
│           │
│           └─ NO: Q: Do you need to reduce memory by sharing common state?
│              │
│              ├─ YES: **FLYWEIGHT PATTERN**
│              │     Example: Text formatting, game particle systems
│              │
│              └─ NO: Q: Do you need to control access to an object?
│                 │
│                 ├─ YES: **PROXY PATTERN**
│                 │     Types:
│                 │     - Virtual Proxy (lazy loading)
│                 │     - Protection Proxy (access control)
│                 │     - Remote Proxy (RMI, microservices)
│                 │     - Cache Proxy (caching layer)
│                 │
│                 └─ NO: Q: Need to decouple abstraction from implementation?
│                    │
│                    ├─ YES: **BRIDGE PATTERN**
│                    │     Example: Device drivers, rendering engines
│                    │
│                    └─ NO: Consider direct composition
```

### Adapter vs Decorator vs Proxy

| Pattern | Intent | Adds Behavior? | Interface Change? |
|---------|--------|----------------|-------------------|
| **Adapter** | Make incompatible interfaces compatible | No | Yes (converts interface) |
| **Decorator** | Add responsibilities dynamically | Yes | No (same interface) |
| **Proxy** | Control access to object | Sometimes | No (same interface) |

**Example Distinction:**
```java
// Adapter - Different interface
interface MediaPlayer { void play(String file); }
interface AdvancedPlayer { void playVlc(String file); }
class MediaAdapter implements MediaPlayer {
    private AdvancedPlayer advancedPlayer;
    public void play(String file) { advancedPlayer.playVlc(file); }
}

// Decorator - Same interface, adds behavior
class EncryptedStream implements InputStream {
    private InputStream wrapped;
    public int read() { return decrypt(wrapped.read()); }
}

// Proxy - Same interface, controls access
class LazyUserProxy implements User {
    private User realUser;
    public String getName() {
        if (realUser == null) realUser = loadFromDB();
        return realUser.getName();
    }
}
```

---

## Behavioral Decisions

### Decision Tree

```
Need to define object interactions/behavior?
│
├─ Q: Do you need to encapsulate requests as objects?
│  │
│  ├─ YES: **COMMAND PATTERN**
│  │     Use cases: Undo/redo, transaction systems, job queues
│  │     Combine with: Memento (for undo), Chain of Responsibility
│  │
│  └─ NO: Q: Do you need to notify multiple objects of state changes?
│     │
│     ├─ YES: **OBSERVER PATTERN**
│     │     Example: Event systems, MVC, reactive programming
│     │     Modern: Use Reactive Streams (Project Reactor, RxJava)
│     │     Combine with: Mediator (for complex notifications)
│     │
│     └─ NO: Q: Do you need to traverse a collection without exposing structure?
│        │
│        ├─ YES: **ITERATOR PATTERN**
│        │     Built-in: Java Collections Framework
│        │     Custom: Complex data structures, lazy evaluation
│        │
│        └─ NO: Q: Do you need to vary algorithm independently?
│           │
│           ├─ YES: **STRATEGY PATTERN**
│           │     Example: Sorting algorithms, payment methods, compression
│           │     Combine with: Factory (to create strategies)
│           │
│           └─ NO: Q: Do you need different behavior based on state?
│              │
│              ├─ YES: **STATE PATTERN**
│              │     Example: TCP connection, order processing, workflow
│              │     Alternative: State machine libraries (Spring State Machine)
│              │
│              └─ NO: Q: Do you need to pass request along chain of handlers?
│                 │
│                 ├─ YES: **CHAIN OF RESPONSIBILITY**
│                 │     Example: Logging, authentication filters, validation
│                 │     Combine with: Command, Decorator
│                 │
│                 └─ NO: Q: Do you need to define operations on object structure?
│                    │
│                    ├─ YES: **VISITOR PATTERN**
│                    │     Example: Compiler AST, document processing
│                    │     ⚠️  WARNING: Makes adding new elements difficult
│                    │
│                    └─ NO: Q: Do you need to define skeleton algorithm?
│                       │
│                       ├─ YES: **TEMPLATE METHOD**
│                       │     Example: Framework hooks, data processing pipelines
│                       │     Alternative: Strategy (more flexible)
│                       │
│                       └─ NO: Q: Need centralized control of interactions?
│                          │
│                          ├─ YES: **MEDIATOR PATTERN**
│                          │     Example: Chat room, UI dialog coordination
│                          │     Combine with: Observer, Command
│                          │
│                          └─ NO: Q: Need to restore object to previous state?
│                             │
│                             ├─ YES: **MEMENTO PATTERN**
│                             │     Example: Undo/redo, snapshots, checkpoints
│                             │     Combine with: Command (undo operations)
│                             │
│                             └─ NO: **INTERPRETER PATTERN**
│                                   Use case: DSL, expression evaluator
│                                   Alternative: Parser generators (ANTLR)
```

### Strategy vs State vs Command

| Pattern | Purpose | When to Use | Example |
|---------|---------|-------------|---------|
| **Strategy** | Interchangeable algorithms | Multiple ways to do same thing | Payment methods, sorting |
| **State** | Behavior changes based on state | Object has distinct states | Order workflow, TCP connection |
| **Command** | Encapsulate request as object | Queue, log, undo operations | Transaction system, macro recording |

---

## Enterprise & Architectural Patterns

### Decision Tree for Distributed Systems

```
Building distributed/enterprise system?
│
├─ Q: Do you need to handle asynchronous communication?
│  │
│  ├─ YES: Q: What's your messaging requirement?
│  │  │
│  │  ├─ Publish/Subscribe: **PUBLISHER-SUBSCRIBER (Kafka, Redis Pub/Sub)**
│  │  │                     Combine with: CQRS, Event Sourcing
│  │  │
│  │  ├─ Point-to-Point: **MESSAGE QUEUE (RabbitMQ, ActiveMQ, IBM MQ)**
│  │  │                  Combine with: Saga, Outbox Pattern
│  │  │
│  │  └─ Request-Reply: **REQUEST-RESPONSE over Message Broker**
│  │                     Combine with: Correlation ID pattern
│  │
│  └─ NO: Q: Do you need to manage distributed transactions?
│     │
│     ├─ YES: **SAGA PATTERN**
│     │     Orchestration: Centralized coordinator
│     │     Choreography: Distributed events
│     │     Combine with: Outbox Pattern, Event Sourcing
│     │     ⚠️  Avoid: Two-Phase Commit (2PC) in microservices
│     │
│     └─ NO: Q: Do you need to separate read/write models?
│        │
│        ├─ YES: **CQRS (Command Query Responsibility Segregation)**
│        │     Combine with: Event Sourcing, Message Queue
│        │     Use case: Different read/write scalability needs
│        │
│        └─ NO: Q: Do you need complete audit trail?
│           │
│           ├─ YES: **EVENT SOURCING**
│           │     Store events instead of current state
│           │     Combine with: CQRS, Snapshot pattern
│           │     Frameworks: Axon, Eventuate
│           │
│           └─ NO: Continue below ↓
```

### Data Access Patterns

```
Working with database?
│
├─ Q: Do you need to abstract database operations?
│  │
│  ├─ YES: **REPOSITORY PATTERN**
│  │     Example: Spring Data JPA repositories
│  │     Combine with: Unit of Work, Specification
│  │
│  └─ NO: Q: Do you need to ensure data consistency without distributed transactions?
│     │
│     ├─ YES: **OUTBOX PATTERN**
│     │     Write to DB + Outbox table in same transaction
│     │     Separate process polls outbox → publishes to message broker
│     │     Combine with: Saga, Event Sourcing
│     │
│     └─ NO: Q: Do you need to handle concurrent updates?
│        │
│        ├─ YES: **OPTIMISTIC LOCKING (version field)**
│        │     Alternative: PESSIMISTIC LOCKING (database locks)
│        │
│        └─ NO: Q: Do you need to coordinate multiple operations?
│           │
│           ├─ YES: **UNIT OF WORK**
│           │     Track changes, commit together
│           │     Built-in: JPA EntityManager
│           │
│           └─ NO: Q: Need to map objects to database?
│              │
│              ├─ YES: **ORM (Hibernate, JPA)**
│              │     Alternative: MyBatis (SQL-focused)
│              │
│              └─ NO: Use JDBC directly
```

### Resilience Patterns

```
Need fault tolerance?
│
├─ **CIRCUIT BREAKER**
│   When: Prevent cascading failures
│   Library: Resilience4j, Hystrix (deprecated)
│   Combine with: Retry, Fallback
│
├─ **RETRY WITH BACKOFF**
│   When: Transient failures expected
│   Types: Exponential backoff, jitter
│   Combine with: Circuit Breaker, Timeout
│
├─ **BULKHEAD**
│   When: Isolate resource pools
│   Example: Separate thread pools per service
│   Combine with: Circuit Breaker
│
├─ **TIMEOUT**
│   When: Prevent indefinite waits
│   Always: Set timeouts on all remote calls
│
└─ **FALLBACK/DEGRADATION**
    When: Need to continue with reduced functionality
    Example: Cached data, default values
    Combine with: Circuit Breaker
```

### Caching Patterns

```
Need to improve performance with caching?
│
├─ **CACHE-ASIDE (Lazy Loading)**
│   App checks cache → miss → load from DB → cache it
│   Best for: Read-heavy workloads
│   
├─ **READ-THROUGH**
│   Cache sits between app and DB, loads automatically
│   Best for: Consistent read patterns
│
├─ **WRITE-THROUGH**
│   Write to cache + DB synchronously
│   Best for: Data consistency critical
│
├─ **WRITE-BEHIND (Write-Back)**
│   Write to cache → async write to DB
│   Best for: Write-heavy, can tolerate eventual consistency
│   ⚠️  Risk: Data loss if cache fails
│
└─ **REFRESH-AHEAD**
    Proactively refresh before expiration
    Best for: Predictable access patterns
```

---

## Combination Patterns

### Common Pattern Combinations

#### 1. **Factory + Strategy + Dependency Injection**
**Use Case:** Flexible algorithm selection with runtime configuration

```java
// Strategy interface
interface PaymentStrategy { PaymentResult process(Payment payment); }

// Concrete strategies
class CreditCardStrategy implements PaymentStrategy { }
class PayPalStrategy implements PaymentStrategy { }
class CryptoStrategy implements PaymentStrategy { }

// Factory to create strategies
@Component
class PaymentStrategyFactory {
    private final Map<PaymentType, PaymentStrategy> strategies;
    
    public PaymentStrategyFactory(List<PaymentStrategy> strategyList) {
        // DI provides all strategies
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                PaymentStrategy::getType,
                Function.identity()
            ));
    }
    
    public PaymentStrategy getStrategy(PaymentType type) {
        return strategies.get(type);
    }
}

// Usage
@Service
class PaymentProcessor {
    private final PaymentStrategyFactory factory;
    
    public PaymentResult process(Payment payment) {
        PaymentStrategy strategy = factory.getStrategy(payment.getType());
        return strategy.process(payment);
    }
}
```

**Why this combination?**
- Factory: Centralizes strategy creation
- Strategy: Encapsulates algorithms
- DI: Makes it testable and flexible

---

#### 2. **Repository + Unit of Work + Specification**
**Use Case:** Complex data access with transaction management

```java
// Repository
interface OrderRepository {
    List<Order> find(Specification<Order> spec);
    void save(Order order);
}

// Specification for complex queries
interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaBuilder cb);
}

class OrdersByCustomerAndStatus implements Specification<Order> {
    private final String customerId;
    private final OrderStatus status;
    
    public Predicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
        return cb.and(
            cb.equal(root.get("customerId"), customerId),
            cb.equal(root.get("status"), status)
        );
    }
}

// Unit of Work (managed by JPA/Spring)
@Transactional
class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;
    
    public void fulfillOrder(String orderId) {
        // All operations in one transaction (Unit of Work)
        Order order = orderRepository.findById(orderId);
        order.setStatus(OrderStatus.PROCESSING);
        
        order.getItems().forEach(item -> 
            inventoryRepository.decrementStock(item.getProductId(), item.getQuantity())
        );
        
        orderRepository.save(order);
        // Transaction commits/rolls back together
    }
}
```

**Why this combination?**
- Repository: Abstracts data access
- Specification: Flexible query building
- Unit of Work: Ensures transaction consistency

---

#### 3. **Builder + Factory Method + Fluent Interface**
**Use Case:** Complex object construction with validation

```java
// Builder with factory method
class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.body = builder.body;
    }
    
    // Factory methods for common cases
    public static Builder newBuilder(String url) {
        return new Builder(url);
    }
    
    public static Builder get(String url) {
        return new Builder(url).method("GET");
    }
    
    public static Builder post(String url) {
        return new Builder(url).method("POST");
    }
    
    // Builder with fluent interface
    public static class Builder {
        private String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body;
        
        private Builder(String url) {
            this.url = Objects.requireNonNull(url);
        }
        
        public Builder method(String method) {
            this.method = method;
            return this;
        }
        
        public Builder header(String key, String value) {
            headers.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public HttpRequest build() {
            validate();
            return new HttpRequest(this);
        }
        
        private void validate() {
            if ("POST".equals(method) && body == null) {
                throw new IllegalStateException("POST requires body");
            }
        }
    }
}

// Usage
HttpRequest request = HttpRequest.post("https://api.example.com/users")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token")
    .body("{\"name\": \"John\"}")
    .build();
```

**Why this combination?**
- Builder: Handles complexity
- Factory Method: Convenience for common cases
- Fluent Interface: Readable API

---

#### 4. **Command + Chain of Responsibility + Template Method**
**Use Case:** Payment processing with validation pipeline

```java
// Command interface
interface PaymentCommand {
    PaymentResult execute(PaymentContext context);
}

// Template method for common flow
abstract class AbstractPaymentCommand implements PaymentCommand {
    
    @Override
    public final PaymentResult execute(PaymentContext context) {
        try {
            validate(context);
            PaymentResult result = doExecute(context);
            logSuccess(context, result);
            return result;
        } catch (Exception e) {
            logFailure(context, e);
            throw e;
        }
    }
    
    protected abstract void validate(PaymentContext context);
    protected abstract PaymentResult doExecute(PaymentContext context);
    
    private void validate(PaymentContext context) { }
    private void logSuccess(PaymentContext context, PaymentResult result) { }
    private void logFailure(PaymentContext context, Exception e) { }
}

// Chain of Responsibility for validation
interface ValidationHandler {
    void validate(PaymentContext context);
    void setNext(ValidationHandler next);
}

class AmountValidationHandler implements ValidationHandler {
    private ValidationHandler next;
    
    public void validate(PaymentContext context) {
        if (context.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidAmountException();
        }
        if (next != null) next.validate(context);
    }
    
    public void setNext(ValidationHandler next) {
        this.next = next;
    }
}

class FraudCheckHandler implements ValidationHandler {
    private ValidationHandler next;
    
    public void validate(PaymentContext context) {
        if (fraudDetectionService.isSuspicious(context)) {
            throw new FraudDetectedException();
        }
        if (next != null) next.validate(context);
    }
    
    public void setNext(ValidationHandler next) {
        this.next = next;
    }
}

// Concrete command
class ProcessCreditCardCommand extends AbstractPaymentCommand {
    
    private final ValidationHandler validationChain;
    
    public ProcessCreditCardCommand() {
        // Build validation chain
        ValidationHandler amount = new AmountValidationHandler();
        ValidationHandler fraud = new FraudCheckHandler();
        ValidationHandler balance = new BalanceCheckHandler();
        
        amount.setNext(fraud);
        fraud.setNext(balance);
        
        this.validationChain = amount;
    }
    
    @Override
    protected void validate(PaymentContext context) {
        validationChain.validate(context);
    }
    
    @Override
    protected PaymentResult doExecute(PaymentContext context) {
        // Process payment
        return paymentGateway.charge(context);
    }
}

// Command executor
class PaymentProcessor {
    
    public PaymentResult process(PaymentCommand command, PaymentContext context) {
        return command.execute(context);
    }
}
```

**Why this combination?**
- Command: Encapsulates payment operation
- Chain of Responsibility: Flexible validation pipeline
- Template Method: Enforces common execution flow

---

#### 5. **Observer + Mediator + Event Bus**
**Use Case:** Complex event system with centralized coordination

```java
// Event
interface DomainEvent {
    String getEventId();
    Instant getTimestamp();
}

class OrderPlacedEvent implements DomainEvent {
    private final String orderId;
    private final String customerId;
    // ... fields, getters
}

// Observer (Listener)
interface EventListener<T extends DomainEvent> {
    void onEvent(T event);
    Class<T> getEventType();
}

@Component
class EmailNotificationListener implements EventListener<OrderPlacedEvent> {
    
    @Override
    public void onEvent(OrderPlacedEvent event) {
        emailService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId());
    }
    
    @Override
    public Class<OrderPlacedEvent> getEventType() {
        return OrderPlacedEvent.class;
    }
}

@Component
class InventoryUpdateListener implements EventListener<OrderPlacedEvent> {
    
    @Override
    public void onEvent(OrderPlacedEvent event) {
        inventoryService.reserveItems(event.getOrderId());
    }
    
    @Override
    public Class<OrderPlacedEvent> getEventType() {
        return OrderPlacedEvent.class;
    }
}

// Mediator (Event Bus)
@Component
class EventBus {
    
    private final Map<Class<? extends DomainEvent>, List<EventListener<?>>> listeners;
    private final ExecutorService executorService;
    
    public EventBus(List<EventListener<?>> listenerList) {
        this.listeners = listenerList.stream()
            .collect(Collectors.groupingBy(EventListener::getEventType));
        this.executorService = Executors.newFixedThreadPool(10);
    }
    
    public <T extends DomainEvent> void publish(T event) {
        List<EventListener<?>> eventListeners = listeners.get(event.getClass());
        
        if (eventListeners != null) {
            eventListeners.forEach(listener -> 
                executorService.submit(() -> {
                    try {
                        ((EventListener<T>) listener).onEvent(event);
                    } catch (Exception e) {
                        // Log error, don't fail publisher
                        logger.error("Event processing failed", e);
                    }
                })
            );
        }
    }
}

// Usage
@Service
class OrderService {
    
    private final EventBus eventBus;
    
    public void placeOrder(Order order) {
        // Business logic
        orderRepository.save(order);
        
        // Publish event
        eventBus.publish(new OrderPlacedEvent(order.getId(), order.getCustomerId()));
    }
}
```

**Why this combination?**
- Observer: Decoupled event handling
- Mediator (EventBus): Centralized event routing
- Async processing: Non-blocking event handling

---

#### 6. **Saga + Outbox + Event Sourcing**
**Use Case:** Distributed transaction with guaranteed delivery

```java
// Event
@Entity
class OutboxEvent {
    @Id
    private String eventId;
    private String aggregateId;
    private String eventType;
    private String payload;
    private Instant createdAt;
    private boolean published;
}

// Aggregate with Event Sourcing
class OrderAggregate {
    private String orderId;
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    public void placeOrder(PlaceOrderCommand command) {
        // Validation
        
        // Create event
        OrderPlacedEvent event = new OrderPlacedEvent(command.orderId);
        apply(event);
        uncommittedEvents.add(event);
    }
    
    public void cancelOrder(CancelOrderCommand command) {
        OrderCancelledEvent event = new OrderCancelledEvent(orderId);
        apply(event);
        uncommittedEvents.add(event);
    }
    
    private void apply(OrderPlacedEvent event) {
        this.orderId = event.getOrderId();
        // Update state
    }
    
    public List<DomainEvent> getUncommittedEvents() {
        return List.copyOf(uncommittedEvents);
    }
}

// Saga Orchestrator
@Service
class OrderSaga {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    
    @Transactional
    public void handleOrderPlaced(OrderPlacedEvent event) {
        try {
            // Step 1: Reserve inventory
            InventoryReservedEvent inventoryEvent = reserveInventory(event);
            saveToOutbox(inventoryEvent);
            
            // Step 2: Process payment
            PaymentProcessedEvent paymentEvent = processPayment(event);
            saveToOutbox(paymentEvent);
            
            // Step 3: Confirm order
            OrderConfirmedEvent confirmedEvent = new OrderConfirmedEvent(event.getOrderId());
            saveToOutbox(confirmedEvent);
            
        } catch (Exception e) {
            // Compensating transactions
            compensate(event);
        }
    }
    
    private void saveToOutbox(DomainEvent event) {
        OutboxEvent outboxEvent = new OutboxEvent(
            event.getEventId(),
            event.getAggregateId(),
            event.getClass().getSimpleName(),
            serializeEvent(event)
        );
        outboxRepository.save(outboxEvent);
    }
    
    private void compensate(OrderPlacedEvent event) {
        // Release inventory
        InventoryReleasedEvent releaseEvent = new InventoryReleasedEvent(event.getOrderId());
        saveToOutbox(releaseEvent);
        
        // Refund payment
        PaymentRefundedEvent refundEvent = new PaymentRefundedEvent(event.getOrderId());
        saveToOutbox(refundEvent);
        
        // Cancel order
        OrderCancelledEvent cancelEvent = new OrderCancelledEvent(event.getOrderId());
        saveToOutbox(cancelEvent);
    }
}

// Outbox Publisher (separate process)
@Component
class OutboxPublisher {
    
    @Scheduled(fixedDelay = 1000)
    public void publishEvents() {
        List<OutboxEvent> unpublished = outboxRepository.findUnpublished();
        
        unpublished.forEach(event -> {
            try {
                kafkaTemplate.send("domain-events", event.getPayload());
                
                event.setPublished(true);
                outboxRepository.save(event);
                
            } catch (Exception e) {
                // Retry on next poll
                logger.error("Failed to publish event", e);
            }
        });
    }
}
```

**Why this combination?**
- Saga: Coordinates distributed transaction
- Outbox: Guarantees at-least-once delivery
- Event Sourcing: Complete audit trail
- Transactional outbox ensures consistency

---

#### 7. **CQRS + Repository + Specification + Cache**
**Use Case:** High-performance read/write separation

```java
// Command Model
@Entity
class Order {
    @Id
    private String id;
    private String customerId;
    private OrderStatus status;
    private List<OrderItem> items;
    // Write-optimized structure
}

interface OrderCommandRepository extends JpaRepository<Order, String> {
    // Write operations only
}

@Service
class OrderCommandService {
    
    private final OrderCommandRepository repository;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public void placeOrder(PlaceOrderCommand command) {
        Order order = new Order(command);
        repository.save(order);
        
        eventPublisher.publish(new OrderPlacedEvent(order));
    }
}

// Query Model (Denormalized for reads)
@Document
class OrderQueryModel {
    private String id;
    private String customerId;
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private List<OrderItemView> items;
    // Read-optimized, denormalized structure
}

interface OrderQueryRepository extends MongoRepository<OrderQueryModel, String> {
    // Read operations only
    @Cacheable("orders")
    List<OrderQueryModel> findByCustomerId(String customerId);
    
    @Cacheable("orders")
    List<OrderQueryModel> findByStatus(OrderStatus status);
}

@Service
class OrderQueryService {
    
    private final OrderQueryRepository repository;
    
    // Specification for complex queries
    public List<OrderQueryModel> search(OrderSearchCriteria criteria) {
        return repository.findAll(
            new OrderSpecification(criteria),
            criteria.getPageable()
        );
    }
    
    @Cacheable(value = "orderSummary", key = "#customerId")
    public OrderSummary getCustomerOrderSummary(String customerId) {
        List<OrderQueryModel> orders = repository.findByCustomerId(customerId);
        return OrderSummary.from(orders);
    }
}

// Event Handler to sync query model
@Component
class OrderQueryModelUpdater {
    
    private final OrderQueryRepository queryRepository;
    
    @EventListener
    public void on(OrderPlacedEvent event) {
        OrderQueryModel queryModel = buildQueryModel(event);
        queryRepository.save(queryModel);
    }
    
    @EventListener
    @CacheEvict(value = "orders", allEntries = true)
    public void on(OrderStatusChangedEvent event) {
        OrderQueryModel model = queryRepository.findById(event.getOrderId())
            .orElseThrow();
        
        model.setStatus(event.getNewStatus());
        queryRepository.save(model);
    }
}
```

**Why this combination?**
- CQRS: Separates read/write concerns
- Repository: Abstracts data access for both sides
- Specification: Flexible query building
- Cache: Performance optimization for reads

---

### Pattern Selection Matrix

| Requirement | Primary Pattern | Combine With | Avoid |
|-------------|----------------|--------------|-------|
| Create complex objects | Builder | Factory Method, DI | Direct instantiation |
| Runtime algorithm selection | Strategy | Factory, DI | Large if-else chains |
| Undo/redo functionality | Command | Memento, Stack | Direct method calls |
| Event notification | Observer | Mediator, Event Bus | Tight coupling |
| Distributed transactions | Saga | Outbox, Event Sourcing | Two-Phase Commit |
| API backward compatibility | Adapter | Facade | Breaking changes |
| Add behavior dynamically | Decorator | Proxy, Factory | Inheritance |
| Validation pipeline | Chain of Responsibility | Command, Strategy | Nested if statements |
| Read/write separation | CQRS | Event Sourcing, Cache | Single model for both |
| Fault tolerance | Circuit Breaker | Retry, Bulkhead | No error handling |

---

## Anti-Pattern Warnings

### ⚠️ When NOT to Use Patterns

#### 1. **Don't Use Singleton**
```
❌ AVOID: Singleton for shared state
✅ USE: Dependency Injection with scope management

Why?
- Makes testing difficult (global state)
- Hides dependencies
- Violates Single Responsibility Principle
- Threading issues

Exception: Truly stateless services (logger configuration)
```

#### 2. **Don't Overuse Abstract Factory**
```
❌ AVOID: Abstract Factory with only one implementation
✅ USE: Simple Factory or direct instantiation

Why?
- Over-engineering for simple cases
- Adds unnecessary complexity
```

#### 3. **Don't Use Visitor Lightly**
```
❌ AVOID: Visitor when object structure changes frequently
✅ USE: Strategy or double dispatch

Why?
- Adding new element types requires changing all visitors
- Breaks Open/Closed Principle
```

#### 4. **Don't Use Template Method for Simple Cases**
```
❌ AVOID: Template Method with single variation point
✅ USE: Strategy Pattern

Why?
- Inheritance-based (less flexible than composition)
- Strategy provides better flexibility
```

#### 5. **Don't Use Observer for Synchronous Calls**
```
❌ AVOID: Observer when you need immediate response
✅ USE: Direct method calls or synchronous events

Why?
- Observer typically async, order not guaranteed
- Harder to debug
```

---

## Quick Reference Cheat Sheet

### By Problem Type

| I need to... | Pattern(s) to Consider |
|-------------|------------------------|
| Create objects with many optional parameters | **Builder** + Static Factory |
| Make incompatible interfaces work | **Adapter** |
| Add behavior without changing class | **Decorator** or **Proxy** |
| Simplify complex subsystem | **Facade** |
| Define interchangeable algorithms | **Strategy** + Factory |
| Handle requests in a pipeline | **Chain of Responsibility** |
| Notify multiple objects of changes | **Observer** + Event Bus |
| Manage distributed transactions | **Saga** + Outbox |
| Separate reads from writes | **CQRS** + Event Sourcing |
| Handle failures gracefully | **Circuit Breaker** + Retry + Fallback |
| Cache frequently accessed data | **Cache-Aside** or **Read-Through** |
| Process commands with undo | **Command** + Memento |
| Traverse complex structures | **Iterator** or **Visitor** |

### Decision Priority Order

```
1. Start with SIMPLEST solution (no pattern)
2. Identify actual problem (not hypothetical)
3. Consider SOLID principles first
4. Choose pattern that solves the problem
5. Combine patterns when needed
6. Refactor when requirements change
```

### Pattern Relationships

```
Composition Patterns:
├─ Decorator wraps objects (same interface)
├─ Adapter converts interfaces
├─ Proxy controls access (same interface)
└─ Facade simplifies interfaces

Behavioral Patterns:
├─ Strategy defines algorithms
├─ State defines state-based behavior
├─ Command encapsulates requests
└─ Chain of Responsibility passes requests

Creation Patterns:
├─ Factory Method delegates creation
├─ Abstract Factory creates families
├─ Builder constructs complex objects
└─ Prototype clones objects
```

---

## Conclusion

### Golden Rules

1. **YAGNI (You Aren't Gonna Need It)**: Don't add patterns speculatively
2. **KISS (Keep It Simple)**: Start simple, refactor when needed
3. **Composition > Inheritance**: Prefer composition-based patterns
4. **Open/Closed Principle**: Patterns should extend, not modify
5. **Dependency Inversion**: Depend on abstractions, not concretions

### When in Doubt

```
1. Does it solve a REAL problem you have NOW?
   NO → Don't use the pattern
   YES → Continue

2. Is there a SIMPLER solution?
   YES → Use the simpler solution
   NO → Continue

3. Will it make code MORE maintainable?
   NO → Don't use the pattern
   YES → Use the pattern

4. Can your team understand and maintain it?
   NO → Reconsider or provide training
   YES → Implement it
```

---

**Remember**: Patterns are tools, not goals. The goal is clean, maintainable, testable code. Use patterns to achieve that goal, not for the sake of using patterns.
