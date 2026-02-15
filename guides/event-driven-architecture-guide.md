# Event-Driven Architecture Mastery

A comprehensive guide to designing, implementing, and operating event-driven systems with sagas, event sourcing, CQRS, and distributed patterns.

---

## Table of Contents

1. [EDA Fundamentals](#eda-fundamentals)
2. [Events vs Commands vs Queries](#events-vs-commands-vs-queries)
3. [Event Design Patterns](#event-design-patterns)
4. [Event Schema Design](#event-schema-design)
5. [Saga Pattern](#saga-pattern)
6. [Event Sourcing](#event-sourcing)
7. [CQRS Pattern](#cqrs-pattern)
8. [Outbox Pattern](#outbox-pattern)
9. [Event Versioning](#event-versioning)
10. [Idempotency](#idempotency)
11. [Event Ordering](#event-ordering)
12. [Error Handling & DLQ](#error-handling--dlq)
13. [Event Replay & Recovery](#event-replay--recovery)
14. [Testing Event-Driven Systems](#testing-event-driven-systems)
15. [Observability](#observability)
16. [Production Patterns](#production-patterns)

---

## EDA Fundamentals

### Why Event-Driven Architecture?

```
Traditional Request-Response:
┌─────────┐          ┌─────────┐          ┌─────────┐
│ Service │ ────────>│ Service │ ────────>│ Service │
│    A    │  Sync    │    B    │  Sync    │    C    │
└─────────┘  Call    └─────────┘  Call    └─────────┘
    ↑                                          │
    └──────────── Coupled & Blocking ──────────┘

Event-Driven:
┌─────────┐          ┌───────────┐
│ Service │ ──event─>│   Event   │
│    A    │  Async   │    Bus    │
└─────────┘          └───────────┘
                       │   │   │
         event────────┘   │   └────────event
           │              │              │
           ▼              ▼              ▼
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │ Service │    │ Service │    │ Service │
      │    B    │    │    C    │    │    D    │
      └─────────┘    └─────────┘    └─────────┘
       Decoupled, Scalable, Resilient
```

### Key Benefits

- **Decoupling** — Services don't know about each other
- **Scalability** — Process events in parallel
- **Resilience** — Failures are isolated
- **Auditability** — Complete event history
- **Flexibility** — Add new consumers without changing producers

---

## Events vs Commands vs Queries

### Event

```java
// Event: Something that HAPPENED (past tense)
@Value
public class PaymentProcessedEvent {
    String paymentId;
    String orderId;
    BigDecimal amount;
    Instant processedAt;
    String userId;
}

// Published by: Payment Service
// Consumed by: Order Service, Notification Service, Analytics Service
```

### Command

```java
// Command: Something to DO (imperative)
@Value
public class ProcessPaymentCommand {
    String paymentId;
    String orderId;
    BigDecimal amount;
    String userId;
}

// Sent by: Order Service
// Handled by: Payment Service (single handler)
```

### Query

```java
// Query: Read-only request
@Value
public class GetPaymentDetailsQuery {
    String paymentId;
}

// Sent by: API Gateway
// Handled by: Payment Query Service (CQRS)
```

### Decision Matrix

| Use Case | Pattern | Example |
|----------|---------|---------|
| Notify multiple systems | Event | `OrderPlaced` → Inventory, Shipping, Email |
| Request specific action | Command | `ProcessPayment` → Payment Service |
| Read data | Query | `GetUserOrders` → Order Query Service |
| State change happened | Event | `UserRegistered`, `InventoryUpdated` |
| Business workflow | Saga | Order → Payment → Shipping (compensating) |

---

## Saga Pattern

### Choreography Saga

```java
// Order Service
@Service
public class OrderService {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void createOrder(CreateOrderRequest request) {
        Order order = Order.create(request);
        orderRepository.save(order);

        // Publish event - no direct calls
        kafkaTemplate.send("order-events",
            new OrderCreatedEvent(
                order.getId(),
                order.getUserId(),
                order.getItems(),
                order.getTotalAmount()
            )
        );
    }

    @KafkaListener(topics = "payment-events")
    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow();

        order.markPaid();
        orderRepository.save(order);

        // Trigger next step
        kafkaTemplate.send("inventory-events",
            new ReserveInventoryEvent(
                order.getId(),
                order.getItems()
            )
        );
    }

    @KafkaListener(topics = "payment-events")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId())
            .orElseThrow();

        order.markCancelled();
        orderRepository.save(order);

        // Compensate
        kafkaTemplate.send("order-events",
            new OrderCancelledEvent(order.getId(), event.getReason())
        );
    }
}
```

### Orchestration Saga

```java
// Saga Orchestrator
@Service
public class OrderSagaOrchestrator {

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private ShippingService shippingService;

    public void executeOrderSaga(Order order) {
        SagaState state = SagaState.create(order.getId());

        try {
            // Step 1: Process payment
            state.addStep("PAYMENT");
            PaymentResult payment = paymentService.processPayment(order);
            state.completeStep("PAYMENT", payment);

            // Step 2: Reserve inventory
            state.addStep("INVENTORY");
            InventoryResult inventory = inventoryService.reserve(order);
            state.completeStep("INVENTORY", inventory);

            // Step 3: Create shipment
            state.addStep("SHIPPING");
            ShipmentResult shipment = shippingService.createShipment(order);
            state.completeStep("SHIPPING", shipment);

            state.complete();

        } catch (Exception e) {
            // Compensate in reverse order
            compensate(state);
        }
    }

    private void compensate(SagaState state) {
        List<String> completedSteps = state.getCompletedSteps();

        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            String step = completedSteps.get(i);

            switch (step) {
                case "SHIPPING" -> shippingService.cancelShipment(state);
                case "INVENTORY" -> inventoryService.releaseReservation(state);
                case "PAYMENT" -> paymentService.refundPayment(state);
            }
        }
    }
}
```

---

## Event Sourcing

### Event Store

```java
@Entity
@Table(name = "event_store")
public class EventStoreEntry {

    @Id
    @GeneratedValue
    private Long id;

    private String aggregateId;
    private String aggregateType;
    private Long version;
    private String eventType;

    @Column(columnDefinition = "jsonb")
    private String eventData;

    private Instant createdAt;
    private String createdBy;
}

@Repository
public interface EventStoreRepository extends JpaRepository<EventStoreEntry, Long> {

    List<EventStoreEntry> findByAggregateIdOrderByVersionAsc(String aggregateId);

    @Query("SELECT MAX(e.version) FROM EventStoreEntry e WHERE e.aggregateId = :aggregateId")
    Optional<Long> findLatestVersion(@Param("aggregateId") String aggregateId);
}
```

### Aggregate with Event Sourcing

```java
public class BankAccount {

    private String accountId;
    private BigDecimal balance;
    private List<Object> uncommittedEvents = new ArrayList<>();

    // Replay events to rebuild state
    public static BankAccount fromEvents(List<Object> events) {
        BankAccount account = new BankAccount();
        events.forEach(account::apply);
        return account;
    }

    // Business method - generates events
    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        MoneyDepositedEvent event = new MoneyDepositedEvent(
            accountId,
            amount,
            Instant.now()
        );

        apply(event);
        uncommittedEvents.add(event);
    }

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) {
            throw new InsufficientFundsException();
        }

        MoneyWithdrawnEvent event = new MoneyWithdrawnEvent(
            accountId,
            amount,
            Instant.now()
        );

        apply(event);
        uncommittedEvents.add(event);
    }

    // Apply event to state (idempotent)
    private void apply(Object event) {
        switch (event) {
            case AccountCreatedEvent e -> {
                this.accountId = e.accountId();
                this.balance = BigDecimal.ZERO;
            }
            case MoneyDepositedEvent e -> {
                this.balance = this.balance.add(e.amount());
            }
            case MoneyWithdrawnEvent e -> {
                this.balance = this.balance.subtract(e.amount());
            }
            default -> throw new IllegalArgumentException(
                "Unknown event: " + event.getClass());
        }
    }

    public List<Object> getUncommittedEvents() {
        return new ArrayList<>(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
}
```

### Event Store Service

```java
@Service
public class EventStoreService {

    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    private final ApplicationEventPublisher eventPublisher;

    public void saveEvents(String aggregateId, List<Object> events, long expectedVersion) {
        // Optimistic locking
        Long currentVersion = repository.findLatestVersion(aggregateId)
            .orElse(0L);

        if (!currentVersion.equals(expectedVersion)) {
            throw new ConcurrencyException(
                "Expected version " + expectedVersion +
                " but was " + currentVersion
            );
        }

        long version = currentVersion;

        for (Object event : events) {
            version++;

            EventStoreEntry entry = new EventStoreEntry();
            entry.setAggregateId(aggregateId);
            entry.setAggregateType("BankAccount");
            entry.setVersion(version);
            entry.setEventType(event.getClass().getSimpleName());
            entry.setEventData(objectMapper.writeValueAsString(event));
            entry.setCreatedAt(Instant.now());

            repository.save(entry);

            // Publish to event bus
            eventPublisher.publishEvent(event);
        }
    }

    public <T> T loadAggregate(String aggregateId, Class<T> aggregateClass) {
        List<EventStoreEntry> entries =
            repository.findByAggregateIdOrderByVersionAsc(aggregateId);

        List<Object> events = entries.stream()
            .map(this::deserializeEvent)
            .toList();

        return (T) BankAccount.fromEvents(events);
    }
}
```

---

## Outbox Pattern

### Database Table

```sql
CREATE TABLE outbox_events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at TIMESTAMP,
    published BOOLEAN NOT NULL DEFAULT FALSE,
    retry_count INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished
    ON outbox_events (published, created_at)
    WHERE published = FALSE;
```

### Transactional Outbox

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public void createOrder(CreateOrderRequest request) {
        // 1. Save business entity
        Order order = Order.create(request);
        orderRepository.save(order);

        // 2. Save event to outbox (same transaction)
        OutboxEvent outboxEvent = OutboxEvent.builder()
            .aggregateId(order.getId())
            .eventType("OrderCreated")
            .payload(toJson(new OrderCreatedEvent(order)))
            .createdAt(Instant.now())
            .published(false)
            .build();

        outboxRepository.save(outboxEvent);

        // Transaction commits - both saved or both rolled back
    }
}
```

### Outbox Publisher

```java
@Service
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000) // Every second
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepository
            .findTop100ByPublishedFalseOrderByCreatedAtAsc();

        for (OutboxEvent event : events) {
            try {
                kafkaTemplate.send(
                    "events",
                    event.getAggregateId(),
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS);

                event.setPublished(true);
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);

            } catch (Exception e) {
                event.setRetryCount(event.getRetryCount() + 1);
                outboxRepository.save(event);

                if (event.getRetryCount() > 10) {
                    // Move to DLQ or alert
                    log.error("Failed to publish event after 10 retries", e);
                }
            }
        }
    }
}
```

---

## Idempotency

### Idempotent Consumer

```java
@Service
public class PaymentEventHandler {

    private final ProcessedEventRepository processedEventRepository;
    private final PaymentRepository paymentRepository;

    @KafkaListener(topics = "payment-events")
    public void handlePaymentEvent(PaymentEvent event,
                                   @Header("messageId") String messageId) {

        // 1. Check if already processed
        if (processedEventRepository.existsById(messageId)) {
            log.info("Event {} already processed, skipping", messageId);
            return;
        }

        // 2. Process event
        Payment payment = processPayment(event);

        // 3. Mark as processed (atomic with business logic)
        ProcessedEvent processedEvent = ProcessedEvent.builder()
            .id(messageId)
            .eventType(event.getClass().getSimpleName())
            .processedAt(Instant.now())
            .build();

        processedEventRepository.save(processedEvent);
    }

    @Transactional
    private Payment processPayment(PaymentEvent event) {
        Payment payment = Payment.create(event);
        return paymentRepository.save(payment);
    }
}
```

### Idempotency Key (API Level)

```java
@PostMapping("/payments")
public ResponseEntity<Payment> createPayment(
        @RequestBody PaymentRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {

    // Check cache first
    Payment cachedPayment = idempotencyCache.get(idempotencyKey);
    if (cachedPayment != null) {
        return ResponseEntity.ok(cachedPayment);
    }

    // Check database
    Payment existingPayment = paymentRepository
        .findByIdempotencyKey(idempotencyKey)
        .orElse(null);

    if (existingPayment != null) {
        return ResponseEntity.ok(existingPayment);
    }

    // Process
    Payment payment = paymentService.process(request, idempotencyKey);

    // Cache for 24 hours
    idempotencyCache.put(idempotencyKey, payment, Duration.ofHours(24));

    return ResponseEntity.status(HttpStatus.CREATED).body(payment);
}
```

---

## Production Patterns

### Dead Letter Queue (DLQ)

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory());

        // Error handler with DLQ
        factory.setCommonErrorHandler(
            new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate(),
                    (record, ex) -> {
                        // Send to DLQ topic
                        return new TopicPartition(
                            record.topic() + ".DLT",
                            record.partition()
                        );
                    }
                ),
                new FixedBackOff(1000L, 3L) // 3 retries with 1s interval
            )
        );

        return factory;
    }
}
```

---

## Best Practices

### ✅ DO

- Design events as immutable facts
- Use event versioning from day one
- Implement idempotent consumers
- Use outbox pattern for transactional integrity
- Include correlation IDs for tracing
- Design for eventual consistency
- Use dead letter queues
- Monitor event lag and processing time
- Test with event replay

### ❌ DON'T

- Include mutable data in events
- Rely on event ordering across partitions
- Ignore idempotency
- Couple services through events
- Store large payloads in events
- Use events for synchronous request-response
- Delete events (they are the source of truth)
- Ignore schema evolution

---

*This guide provides comprehensive patterns for building production-grade event-driven systems. Combine these patterns based on your specific requirements and constraints.*
