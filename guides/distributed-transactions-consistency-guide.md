# Distributed Transactions & Consistency Patterns

A comprehensive guide to managing distributed transactions, consistency patterns, and data integrity in microservices architectures with practical implementations.

---

## Table of Contents

1. [Fundamentals](#fundamentals)
2. [CAP Theorem in Practice](#cap-theorem-in-practice)
3. [Consistency Models](#consistency-models)
4. [Two-Phase Commit (2PC)](#two-phase-commit-2pc)
5. [Saga Pattern](#saga-pattern)
6. [Outbox Pattern](#outbox-pattern)
7. [Idempotency](#idempotency)
8. [Eventual Consistency](#eventual-consistency)
9. [Compensating Transactions](#compensating-transactions)
10. [Event Sourcing for Consistency](#event-sourcing-for-consistency)
11. [Production Patterns](#production-patterns)

---

## Fundamentals

### The Distributed Transaction Problem

```
Monolithic Application:
┌────────────────────────────┐
│    Single Database         │
│                            │
│  BEGIN TRANSACTION;        │
│  UPDATE orders...          │
│  UPDATE inventory...       │
│  UPDATE payments...        │
│  COMMIT;                   │
│                            │
│  ✅ ACID guaranteed        │
└────────────────────────────┘

Microservices:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Order      │  │  Inventory   │  │   Payment    │
│   Service    │  │   Service    │  │   Service    │
│     DB       │  │     DB       │  │     DB       │
└──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
       └─────────────────┴─────────────────┘
              ❌ No distributed ACID
              ✅ Need coordination patterns
```

### Why Distributed Transactions Are Hard

```
Problems:
1. Network Failures
   - Service unreachable
   - Timeout during commit
   - Partial failures

2. Service Failures
   - Crash during transaction
   - Database unavailable
   - Out of memory

3. Performance
   - Locks across services
   - Increased latency
   - Reduced throughput

4. Coupling
   - Services must coordinate
   - Tight coupling
   - Availability impact
```

---

## CAP Theorem in Practice

### Understanding CAP

```
CAP Theorem: You can have at most 2 of 3:
- Consistency: All nodes see same data
- Availability: Every request gets response
- Partition Tolerance: System works despite network failures

Real World:
- Network partitions WILL happen
- Must choose: CP or AP

CP (Consistency + Partition Tolerance):
- Banking systems
- Inventory management
- Strong consistency needed
- Examples: MongoDB, HBase

AP (Availability + Partition Tolerance):
- Social media feeds
- Analytics
- Eventual consistency acceptable
- Examples: Cassandra, DynamoDB
```

### Practical Trade-offs

```java
// CP System: Reject requests during partition
@Service
public class InventoryService {

    private final DistributedLock lock;

    public void reserveItem(String itemId, int quantity) {
        // Get distributed lock (CP choice)
        if (!lock.tryLock(itemId, Duration.ofSeconds(5))) {
            throw new ServiceUnavailableException(
                "Cannot acquire lock - possible partition"
            );
        }

        try {
            // Strong consistency guaranteed
            inventory.reserve(itemId, quantity);
        } finally {
            lock.unlock(itemId);
        }
    }
}

// AP System: Always available, eventual consistency
@Service
public class ViewCountService {

    private final AsyncEventPublisher publisher;

    public void incrementViewCount(String postId) {
        // Always accept (AP choice)
        // Eventual consistency
        publisher.publish(new ViewIncrementedEvent(postId));

        // No blocking, always available
        // Counts will converge eventually
    }
}
```

---

## Consistency Models

### Consistency Spectrum

```
Strong Consistency:
- Linearizability
- All reads see latest write
- Highest cost, lowest availability

Sequential Consistency:
- Operations appear in program order
- May not see latest immediately

Causal Consistency:
- Causally related operations ordered
- Concurrent operations may differ

Eventual Consistency:
- All replicas converge eventually
- Lowest cost, highest availability
- No order guarantees
```

### Implementation Examples

```java
// Strong Consistency with Distributed Lock
@Service
public class AccountService {

    private final RedissonClient redisson;
    private final AccountRepository repository;

    @Transactional
    public void transfer(String fromId, String toId, Money amount) {
        RLock lock1 = redisson.getLock("account:" + fromId);
        RLock lock2 = redisson.getLock("account:" + toId);

        // Multi-lock for deadlock prevention
        RLock multiLock = redisson.getMultiLock(lock1, lock2);

        try {
            multiLock.lock(10, TimeUnit.SECONDS);

            Account from = repository.findById(fromId)
                .orElseThrow();
            Account to = repository.findById(toId)
                .orElseThrow();

            from.debit(amount);
            to.credit(amount);

            repository.saveAll(List.of(from, to));

        } finally {
            multiLock.unlock();
        }
    }
}

// Eventual Consistency with Event Sourcing
@Service
public class OrderProjectionService {

    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderEvent event) {
        // Eventually consistent read model
        // Multiple replicas may be temporarily inconsistent

        switch (event.getType()) {
            case ORDER_CREATED ->
                createOrderProjection(event);
            case ORDER_CONFIRMED ->
                updateOrderStatus(event);
            case ORDER_SHIPPED ->
                updateShippingInfo(event);
        }

        // Eventually all replicas converge
    }
}
```

---

## Two-Phase Commit (2PC)

### How 2PC Works

```
Coordinator:                 Participants:
    │                        │     │     │
    │  1. PREPARE           │     │     │
    ├────────────────────────►     │     │
    │                        │     │     │
    │  2. Vote: YES/NO      │     │     │
    ◄────────────────────────┤     │     │
    │                        │     │     │
    │  3. COMMIT/ABORT      │     │     │
    ├────────────────────────►─────►─────►
    │                        │     │     │
    │  4. ACK               │     │     │
    ◄────────────────────────┴─────┴─────┘
```

### 2PC Implementation

```java
// Coordinator
@Service
public class TwoPhaseCommitCoordinator {

    private final List<TransactionalService> participants;
    private final TransactionLog transactionLog;

    public void executeTransaction(DistributedTransaction tx) {
        String txId = UUID.randomUUID().toString();

        try {
            // Phase 1: PREPARE
            transactionLog.log(txId, TransactionPhase.PREPARE);

            List<PrepareResponse> responses = participants.stream()
                .map(p -> p.prepare(txId, tx))
                .toList();

            boolean allVotedYes = responses.stream()
                .allMatch(PrepareResponse::isYes);

            if (allVotedYes) {
                // Phase 2: COMMIT
                transactionLog.log(txId, TransactionPhase.COMMIT);

                participants.forEach(p -> p.commit(txId));

            } else {
                // Phase 2: ABORT
                transactionLog.log(txId, TransactionPhase.ABORT);

                participants.forEach(p -> p.abort(txId));
            }

        } catch (Exception e) {
            // Abort on any error
            transactionLog.log(txId, TransactionPhase.ABORT);
            participants.forEach(p -> p.abort(txId));
            throw new TransactionFailedException(txId, e);
        }
    }
}

// Participant
@Service
public class PaymentService implements TransactionalService {

    private final Map<String, PreparedTransaction> prepared =
        new ConcurrentHashMap<>();

    @Override
    public PrepareResponse prepare(String txId, DistributedTransaction tx) {
        try {
            // Validate and lock resources
            Payment payment = createPayment(tx.getPaymentData());

            if (canAuthorize(payment)) {
                // Lock resources
                prepared.put(txId, new PreparedTransaction(payment));
                return PrepareResponse.yes();
            } else {
                return PrepareResponse.no("Insufficient funds");
            }

        } catch (Exception e) {
            return PrepareResponse.no(e.getMessage());
        }
    }

    @Override
    public void commit(String txId) {
        PreparedTransaction prepared = this.prepared.remove(txId);

        if (prepared != null) {
            // Perform actual work
            paymentRepository.save(prepared.getPayment());
        }
    }

    @Override
    public void abort(String txId) {
        // Release locks
        prepared.remove(txId);
    }
}
```

### 2PC Problems

```
Issues:
1. Blocking Protocol
   - Participants wait for coordinator
   - Poor performance

2. Single Point of Failure
   - Coordinator crash = blocked participants
   - Need coordinator recovery

3. Performance
   - 2 network round trips
   - Holding locks during protocol

4. Availability
   - Any participant failure = abort
   - Reduces overall availability

Recommendation:
❌ Avoid 2PC in microservices
✅ Use Saga pattern instead
```

---

## Saga Pattern

### Choreography-Based Saga

```java
// Order Service initiates saga
@Service
public class OrderService {

    private final OrderRepository repository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Step 1: Create order
        Order order = Order.create(request);
        order.setStatus(OrderStatus.PENDING);
        repository.save(order);

        // Trigger saga
        eventPublisher.publishEvent(
            new OrderCreatedEvent(order.getId(), order.getItems())
        );

        return order;
    }

    // Compensation
    @TransactionalEventListener
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Order order = repository.findById(event.getOrderId())
            .orElseThrow();

        order.setStatus(OrderStatus.CANCELLED);
        repository.save(order);

        eventPublisher.publishEvent(
            new OrderCancelledEvent(order.getId())
        );
    }
}

// Inventory Service
@Service
public class InventoryService {

    @TransactionalEventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Step 2: Reserve inventory
            inventory.reserve(event.getItems());

            eventPublisher.publishEvent(
                new InventoryReservedEvent(event.getOrderId())
            );

        } catch (InsufficientInventoryException e) {
            eventPublisher.publishEvent(
                new InventoryReservationFailedEvent(event.getOrderId())
            );
        }
    }

    // Compensation
    @TransactionalEventListener
    public void handleOrderCancelled(OrderCancelledEvent event) {
        inventory.release(event.getOrderId());
    }
}

// Payment Service
@Service
public class PaymentService {

    @TransactionalEventListener
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            // Step 3: Process payment
            Payment payment = processPayment(event.getOrderId());

            eventPublisher.publishEvent(
                new PaymentCompletedEvent(event.getOrderId(), payment.getId())
            );

        } catch (PaymentException e) {
            eventPublisher.publishEvent(
                new PaymentFailedEvent(event.getOrderId(), e.getMessage())
            );
        }
    }
}
```

### Orchestration-Based Saga

```java
// Saga Orchestrator
@Service
public class OrderSagaOrchestrator {

    private final SagaStateRepository sagaRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;

    public void executeOrderSaga(OrderId orderId) {
        SagaState saga = SagaState.start(orderId);
        sagaRepository.save(saga);

        try {
            // Step 1: Reserve inventory
            saga.recordStep(SagaStep.INVENTORY_RESERVATION);
            inventoryService.reserve(orderId);
            saga.completeStep(SagaStep.INVENTORY_RESERVATION);

            // Step 2: Process payment
            saga.recordStep(SagaStep.PAYMENT);
            PaymentResult result = paymentService.charge(orderId);
            saga.completeStep(SagaStep.PAYMENT);

            // Step 3: Create shipment
            saga.recordStep(SagaStep.SHIPMENT);
            shippingService.createShipment(orderId);
            saga.completeStep(SagaStep.SHIPMENT);

            saga.markComplete();
            sagaRepository.save(saga);

        } catch (Exception e) {
            // Compensate in reverse order
            compensate(saga);
            throw new SagaFailedException(orderId, e);
        }
    }

    private void compensate(SagaState saga) {
        List<SagaStep> completedSteps = saga.getCompletedSteps();

        // Reverse order compensation
        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);

            try {
                switch (step) {
                    case SHIPMENT ->
                        shippingService.cancelShipment(saga.getOrderId());
                    case PAYMENT ->
                        paymentService.refund(saga.getOrderId());
                    case INVENTORY_RESERVATION ->
                        inventoryService.release(saga.getOrderId());
                }

                saga.recordCompensation(step);

            } catch (Exception e) {
                log.error("Compensation failed for step: {}", step, e);
                saga.recordCompensationFailure(step, e.getMessage());
            }
        }

        saga.markCompensated();
        sagaRepository.save(saga);
    }
}

// Saga State
@Document(collection = "saga_state")
public class SagaState {
    private String id;
    private OrderId orderId;
    private List<SagaStep> completedSteps = new ArrayList<>();
    private List<SagaStep> compensatedSteps = new ArrayList<>();
    private SagaStatus status;
    private Instant startedAt;
    private Instant completedAt;

    public void recordStep(SagaStep step) {
        log.info("Starting saga step: {} for order: {}", step, orderId);
    }

    public void completeStep(SagaStep step) {
        completedSteps.add(step);
        log.info("Completed saga step: {} for order: {}", step, orderId);
    }
}
```

---

## Outbox Pattern

### Implementation

```java
// Outbox Entity
@Entity
@Table(name = "outbox")
public class OutboxEvent {

    @Id
    private String id;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false, columnDefinition = "jsonb")
    private String payload;

    @Column(nullable = false)
    private Instant createdAt;

    @Column
    private Instant publishedAt;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;
}

// Service with Outbox
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Business logic
        Order order = Order.create(request);
        orderRepository.save(order);

        // Save to outbox in same transaction
        OutboxEvent event = OutboxEvent.builder()
            .id(UUID.randomUUID().toString())
            .aggregateId(order.getId().getValue())
            .eventType("OrderCreated")
            .payload(toJson(new OrderCreatedEvent(order)))
            .createdAt(Instant.now())
            .status(OutboxStatus.PENDING)
            .build();

        outboxRepository.save(event);

        // Commit happens atomically
        return order;
    }
}

// Outbox Publisher
@Service
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository
            .findByStatusOrderByCreatedAt(OutboxStatus.PENDING,
                PageRequest.of(0, 100));

        for (OutboxEvent event : pending) {
            try {
                // Publish to Kafka
                kafkaTemplate.send(
                    event.getEventType(),
                    event.getAggregateId(),
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS);

                // Mark as published
                event.setPublishedAt(Instant.now());
                event.setStatus(OutboxStatus.PUBLISHED);
                outboxRepository.save(event);

            } catch (Exception e) {
                log.error("Failed to publish event: {}", event.getId(), e);

                event.setStatus(OutboxStatus.FAILED);
                outboxRepository.save(event);
            }
        }
    }
}

// CDC-Based Outbox with Debezium
// debezium-connector.json
{
  "name": "order-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "secret",
    "database.dbname": "orders",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregateId",
    "transforms.outbox.table.field.event.type": "eventType",
    "transforms.outbox.table.field.event.payload": "payload"
  }
}
```

---

## Idempotency

### Idempotent API

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final PaymentService paymentService;

    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody CreatePaymentRequest request) {

        PaymentResponse response = paymentService.processPayment(
            idempotencyKey,
            request
        );

        return ResponseEntity.ok(response);
    }
}

@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final IdempotencyRepository idempotencyRepository;

    @Transactional
    public PaymentResponse processPayment(
            String idempotencyKey,
            CreatePaymentRequest request) {

        // Check if already processed
        Optional<IdempotencyRecord> existing =
            idempotencyRepository.findById(idempotencyKey);

        if (existing.isPresent()) {
            // Return cached response
            return existing.get().getResponse();
        }

        // Process payment
        Payment payment = Payment.create(request);
        payment = paymentRepository.save(payment);

        PaymentResponse response = toResponse(payment);

        // Store idempotency record
        IdempotencyRecord record = IdempotencyRecord.builder()
            .idempotencyKey(idempotencyKey)
            .response(response)
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
            .build();

        idempotencyRepository.save(record);

        return response;
    }
}

// Idempotent Event Handler
@Service
public class OrderEventHandler {

    private final ProcessedEventRepository processedEventRepository;

    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderEvent event) {
        String eventId = event.getId();

        // Check if already processed
        if (processedEventRepository.existsById(eventId)) {
            log.info("Event already processed: {}", eventId);
            return; // Idempotent - skip duplicate
        }

        try {
            // Process event
            processOrder(event);

            // Mark as processed
            ProcessedEvent processed = ProcessedEvent.builder()
                .eventId(eventId)
                .eventType(event.getType())
                .processedAt(Instant.now())
                .build();

            processedEventRepository.save(processed);

        } catch (Exception e) {
            log.error("Failed to process event: {}", eventId, e);
            throw e; // Retry
        }
    }
}
```

---

## Production Patterns

### Timeout and Retry

```java
@Configuration
public class ResilienceConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(100, 2.0, 10000)
            .retryOn(TransientException.class)
            .build();
    }

    @Bean
    public TimeLimiter timeLimiter() {
        return TimeLimiter.of(
            TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(5))
                .build()
        );
    }
}

@Service
public class ResilientPaymentService {

    private final RetryTemplate retryTemplate;
    private final PaymentGateway gateway;

    public PaymentResult charge(Payment payment) {
        return retryTemplate.execute(context -> {
            try {
                return gateway.charge(payment);
            } catch (NetworkException e) {
                log.warn("Payment attempt {} failed",
                    context.getRetryCount());
                throw e; // Retry
            }
        });
    }
}
```

### Monitoring and Alerting

```java
@Service
public class SagaMonitoringService {

    private final MeterRegistry meterRegistry;

    public void recordSagaStart(String sagaType) {
        meterRegistry.counter("saga.started",
            "type", sagaType).increment();
    }

    public void recordSagaSuccess(String sagaType, Duration duration) {
        meterRegistry.counter("saga.completed",
            "type", sagaType, "status", "success").increment();

        meterRegistry.timer("saga.duration",
            "type", sagaType).record(duration);
    }

    public void recordSagaFailure(String sagaType, String reason) {
        meterRegistry.counter("saga.completed",
            "type", sagaType, "status", "failed").increment();

        meterRegistry.counter("saga.failures",
            "type", sagaType, "reason", reason).increment();
    }

    public void recordCompensation(String sagaType) {
        meterRegistry.counter("saga.compensated",
            "type", sagaType).increment();
    }
}
```

---

## Best Practices

### ✅ DO

- Use Saga pattern for distributed transactions
- Implement idempotency for all operations
- Use Outbox pattern for reliable event publishing
- Design compensating transactions upfront
- Monitor saga execution and failures
- Implement proper timeout handling
- Use correlation IDs for tracing
- Test compensation logic thoroughly
- Accept eventual consistency when possible
- Document consistency guarantees

### ❌ DON'T

- Use distributed 2PC in microservices
- Assume immediate consistency
- Ignore partial failures
- Skip idempotency checks
- Forget about compensation logic
- Use cascading synchronous calls
- Hold locks across services
- Ignore saga state persistence
- Skip monitoring and alerting
- Assume network is reliable

---

*This guide provides production-ready patterns for distributed transactions and consistency. Remember: in distributed systems, eventual consistency is often the right choice over distributed ACID transactions.*
