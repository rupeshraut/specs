# ⚡ Event-Driven Architecture Patterns Cheat Sheet

> **Purpose:** Production-grade patterns for designing, implementing, and operating event-driven systems. Reference before designing any event flow, saga, or eventual consistency mechanism.
> **Stack context:** Java 21+ / Spring Boot 3.x / Kafka / MongoDB / ActiveMQ Artemis / Resilience4j

---

## 📋 The EDA Decision Framework

Before introducing any event-driven pattern, answer:

| Question | Determines |
|----------|-----------|
| Does the caller **need a response**? | Sync (request-reply) vs async (fire-and-forget) |
| Must actions happen **atomically**? | Saga vs distributed transaction vs outbox |
| Is **ordering** critical? | Partitioning strategy, single-writer |
| How **eventual** can consistency be? | Seconds? Minutes? Never acceptable? |
| Who needs to **react** to this? (known or unknown) | Command vs event, point-to-point vs pub-sub |
| What if the event is **processed twice**? | Idempotency strategy |
| What if the event is **never processed**? | Dead-letter, monitoring, SLA |
| Do I need to **reconstruct past state**? | Event sourcing vs state-based |
| How will **schemas evolve** over time? | Versioning strategy |

---

## 🗺️ Event-Driven Topology Map

```
                        ┌─────────────────────────────────────────────┐
                        │          EVENT-DRIVEN PATTERNS              │
                        └──────────────────┬──────────────────────────┘
                                           │
              ┌────────────────────────────┼─────────────────────────────┐
              │                            │                             │
     ┌────────▼────────┐        ┌─────────▼──────────┐       ┌─────────▼──────────┐
     │  Communication   │        │   Consistency       │       │   Data              │
     │  Patterns        │        │   Patterns          │       │   Patterns          │
     └────────┬────────┘        └─────────┬──────────┘       └─────────┬──────────┘
              │                            │                             │
     • Event Notification          • Saga (Orchestration)        • Event Sourcing
     • Event-Carried State         • Saga (Choreography)         • CQRS
     • Command Message             • Outbox Pattern              • Event Store
     • Request-Reply               • Change Data Capture         • Projections
     • Pub-Sub vs P2P              • Idempotent Consumer         • Snapshots
     • Event Mesh                  • Compensating Transaction    • Temporal Queries
```

---

## 📨 Pattern 1: Event Types & Design

### Three Fundamental Event Types

```
1. EVENT NOTIFICATION (Thin Event)
   ─────────────────────────────────
   "Something happened — look it up if you care."
   
   Payload:  { eventId, type, aggregateId, timestamp }
   Size:     Tiny (< 1KB)
   Coupling: Loose — consumer must call back for details
   
   Use when: Many unknown consumers, minimize payload coupling
   Example:  "Payment pay-123 was completed"


2. EVENT-CARRIED STATE TRANSFER (Fat Event)
   ─────────────────────────────────
   "Something happened — here's everything you need."
   
   Payload:  { eventId, type, aggregateId, timestamp, fullEntityState }
   Size:     Larger (1-100KB)
   Coupling: Tighter schema coupling, but no callbacks
   
   Use when: Consumer needs data to proceed, avoid synchronous lookups
   Example:  "Payment pay-123 completed: {amount: 100, currency: USD, customer: {...}}"


3. DOMAIN EVENT (Business Fact)
   ─────────────────────────────────
   "A business-meaningful thing occurred."
   
   Payload:  { eventId, type, aggregateId, timestamp, businessRelevantFields }
   Size:     Medium — only business-relevant data
   Coupling: Moderate — reflects domain language
   
   Use when: Modeling business processes, event sourcing
   Example:  "PaymentCaptured: {paymentId, settledAmount, settlementId, capturedAt}"
```

### Event vs Command

```
EVENT                                    COMMAND
──────────────────────                   ──────────────────────
Past tense: "PaymentCompleted"           Imperative: "ProcessPayment"
Broadcast: anyone can subscribe          Directed: one specific handler
No expectation of action                 Expects action/response
Publisher doesn't know consumers         Sender knows the receiver
Can't be rejected (already happened)     Can be rejected/failed
Multiple consumers OK                    Single consumer expected

When to use EVENT:                       When to use COMMAND:
  • Decoupling services                    • Orchestrated workflows
  • Audit trails                           • Request-reply patterns
  • Unknown/future consumers               • Explicit error handling needed
  • Analytics, notifications               • Sequenced operations
```

### Event Envelope — Standard Schema

```java
// Every event wraps in this envelope
public record EventEnvelope<T>(
    // ── Identity ──
    String eventId,              // UUID — globally unique
    String eventType,            // "payment.captured" — dot-notation
    int schemaVersion,           // For evolution (1, 2, 3...)

    // ── Aggregate context ──
    String aggregateId,          // Entity this belongs to (paymentId)
    String aggregateType,        // "Payment", "Order"
    long aggregateVersion,       // Optimistic concurrency for event sourcing

    // ── Tracing ──
    String correlationId,        // Trace across entire flow
    String causationId,          // Direct parent event/command that caused this

    // ── Metadata ──
    Instant timestamp,           // When the event occurred (not when published)
    String source,               // "payment-service", "fraud-service"
    Map<String, String> headers, // Extensible metadata

    // ── Payload ──
    T payload                    // The actual domain event data
) {
    public EventEnvelope {
        Objects.requireNonNull(eventId);
        Objects.requireNonNull(eventType);
        Objects.requireNonNull(aggregateId);
        Objects.requireNonNull(timestamp);
        if (schemaVersion < 1) throw new IllegalArgumentException("schemaVersion must be >= 1");
    }

    // Factory method with sensible defaults
    public static <T> EventEnvelope<T> wrap(String aggregateId, String aggregateType, T payload) {
        return new EventEnvelope<>(
            UUID.randomUUID().toString(),
            payload.getClass().getSimpleName().replaceAll("([a-z])([A-Z])", "$1.$2").toLowerCase(),
            1,
            aggregateId, aggregateType, 0,
            MDC.get("correlationId"), MDC.get("causationId"),
            Instant.now(), ServiceIdentity.name(),
            Map.of(),
            payload
        );
    }
}
```

### Sealed Event Hierarchy — Domain Events

```java
public sealed interface PaymentEvent {
    String paymentId();
    Instant occurredAt();

    // ── Lifecycle events ──
    record Initiated(
        String paymentId, Instant occurredAt,
        Money amount, String customerId, PaymentMethod method,
        String idempotencyKey
    ) implements PaymentEvent {}

    record Validated(
        String paymentId, Instant occurredAt,
        ValidationResult result
    ) implements PaymentEvent {}

    record Authorized(
        String paymentId, Instant occurredAt,
        String authorizationCode, Instant expiresAt
    ) implements PaymentEvent {}

    record Captured(
        String paymentId, Instant occurredAt,
        Money settledAmount, String settlementId,
        Money fee
    ) implements PaymentEvent {}

    record Failed(
        String paymentId, Instant occurredAt,
        String errorCode, String reason, boolean retryable
    ) implements PaymentEvent {}

    record Refunded(
        String paymentId, Instant occurredAt,
        Money refundAmount, String refundId, String reason
    ) implements PaymentEvent {}

    // ── Marker for terminal states ──
    default boolean isTerminal() {
        return switch (this) {
            case Captured _, Failed f when !f.retryable(), Refunded _ -> true;
            default -> false;
        };
    }
}
```

---

## 🔄 Pattern 2: Saga — Orchestration

> **When:** Multi-step business process across services where atomicity is needed.
> **Approach:** Central orchestrator controls step execution and compensation.

### Saga State Machine

```
                    ┌──────────┐
          ┌────────►│ STARTED  │
          │         └────┬─────┘
          │              │ reserveFunds()
          │              ▼
          │         ┌──────────────┐
          │    ┌────│FUNDS_RESERVED│
          │    │    └────┬─────────┘
          │    │         │ checkFraud()
          │    │         ▼
          │    │    ┌──────────────┐
          │    │ ┌──│FRAUD_CLEARED │
          │    │ │  └────┬─────────┘
          │    │ │       │ chargeGateway()
          │    │ │       ▼
          │    │ │  ┌──────────────┐
          │    │ │  │   CAPTURED   │──────► COMPLETED ✅
          │    │ │  └──────────────┘
          │    │ │
   COMPENSATING│ │  On failure at any step:
          │    │ │       │
          │    │ │       ▼
          │    │ │  ┌──────────────┐
          │    │ └─►│COMPENSATING  │ ← Runs compensations in reverse
          │    └───►│              │
          │         └────┬─────────┘
          │              │
          │              ▼
          │         ┌──────────────┐
          └─────────│   FAILED     │ ✗
                    └──────────────┘
```

### Generic Saga Orchestrator

```java
// ── Step definition ──
public record SagaStep<T>(
    String name,
    Function<SagaContext, T> action,
    Consumer<SagaContext> compensation,
    Predicate<Exception> retryable
) {
    public static <T> SagaStep<T> of(
            String name,
            Function<SagaContext, T> action,
            Consumer<SagaContext> compensation) {
        return new SagaStep<>(name, action, compensation, e -> false);
    }

    public SagaStep<T> withRetry(Predicate<Exception> retryable) {
        return new SagaStep<>(name, action, compensation, retryable);
    }
}

// ── Saga context — shared state across steps ──
public class SagaContext {
    private final Map<String, Object> data = new ConcurrentHashMap<>();
    private final String sagaId;
    private final String correlationId;

    @SuppressWarnings("unchecked")
    public <T> T get(String key) { return (T) data.get(key); }
    public void put(String key, Object value) { data.put(key, value); }
}

// ── Orchestrator ──
@Component
public class SagaOrchestrator {

    private final MeterRegistry meterRegistry;
    private final SagaRepository sagaRepository;  // Persists saga state for recovery

    public <T> SagaResult execute(String sagaName, SagaContext context, List<SagaStep<?>> steps) {
        var completedSteps = new ArrayDeque<SagaStep<?>>();
        var sagaId = context.sagaId();

        sagaRepository.save(new SagaRecord(sagaId, sagaName, SagaStatus.STARTED, Instant.now()));

        for (var step : steps) {
            try {
                log.info("Saga [{}] executing step: {}", sagaId, step.name());
                sagaRepository.updateStep(sagaId, step.name(), StepStatus.EXECUTING);

                var result = executeWithRetry(step, context);
                context.put(step.name() + ".result", result);
                completedSteps.push(step);

                sagaRepository.updateStep(sagaId, step.name(), StepStatus.COMPLETED);
                meterRegistry.counter("saga.step.success",
                    "saga", sagaName, "step", step.name()).increment();

            } catch (Exception e) {
                log.error("Saga [{}] failed at step {}: {}", sagaId, step.name(), e.getMessage());
                sagaRepository.updateStep(sagaId, step.name(), StepStatus.FAILED);
                meterRegistry.counter("saga.step.failure",
                    "saga", sagaName, "step", step.name()).increment();

                compensate(sagaId, sagaName, context, completedSteps);
                sagaRepository.updateStatus(sagaId, SagaStatus.COMPENSATED);
                return SagaResult.failed(step.name(), e);
            }
        }

        sagaRepository.updateStatus(sagaId, SagaStatus.COMPLETED);
        meterRegistry.counter("saga.completed", "saga", sagaName).increment();
        return SagaResult.success(context);
    }

    private <T> T executeWithRetry(SagaStep<T> step, SagaContext context) {
        var retry = Retry.of(step.name(), RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryOnException(step.retryable())
            .build());

        return Retry.decorateSupplier(retry, () -> step.action().apply(context)).get();
    }

    private void compensate(String sagaId, String sagaName,
                           SagaContext context, Deque<SagaStep<?>> completedSteps) {
        log.warn("Saga [{}] compensating {} steps", sagaId, completedSteps.size());
        sagaRepository.updateStatus(sagaId, SagaStatus.COMPENSATING);

        while (!completedSteps.isEmpty()) {
            var step = completedSteps.pop();
            try {
                log.info("Saga [{}] compensating step: {}", sagaId, step.name());
                step.compensation().accept(context);
                sagaRepository.updateStep(sagaId, step.name(), StepStatus.COMPENSATED);
            } catch (Exception e) {
                log.error("CRITICAL: Saga [{}] compensation failed at step {}: {}",
                    sagaId, step.name(), e.getMessage(), e);
                sagaRepository.updateStep(sagaId, step.name(), StepStatus.COMPENSATION_FAILED);
                meterRegistry.counter("saga.compensation.failure",
                    "saga", sagaName, "step", step.name()).increment();
                // Continue compensating remaining steps — don't stop
            }
        }
    }
}
```

### Payment Saga — Concrete Implementation

```java
@Component
public class PaymentSaga {

    private final SagaOrchestrator orchestrator;
    private final AccountService accountService;
    private final FraudService fraudService;
    private final PaymentGateway gateway;
    private final NotificationService notifications;

    public SagaResult processPayment(PaymentRequest request) {
        var context = new SagaContext(UUID.randomUUID().toString(), request.correlationId());
        context.put("request", request);

        var steps = List.<SagaStep<?>>of(
            // Step 1: Reserve funds
            SagaStep.of("reserve-funds",
                ctx -> {
                    var req = ctx.<PaymentRequest>get("request");
                    var reservation = accountService.reserveFunds(req.accountId(), req.amount());
                    ctx.put("reservationId", reservation.id());
                    return reservation;
                },
                ctx -> accountService.releaseReservation(ctx.get("reservationId"))
            ).withRetry(e -> e instanceof ConnectException),

            // Step 2: Fraud check
            SagaStep.of("fraud-check",
                ctx -> {
                    var req = ctx.<PaymentRequest>get("request");
                    var result = fraudService.evaluate(req);
                    if (!result.approved()) {
                        throw new FraudRejectedException(result.reason());
                    }
                    ctx.put("fraudCheckId", result.checkId());
                    return result;
                },
                ctx -> fraudService.clearFlag(ctx.get("fraudCheckId"))
            ),

            // Step 3: Charge payment gateway
            SagaStep.of("charge-gateway",
                ctx -> {
                    var req = ctx.<PaymentRequest>get("request");
                    var charge = gateway.charge(req);
                    ctx.put("transactionId", charge.transactionId());
                    return charge;
                },
                ctx -> gateway.refund(ctx.get("transactionId"))
            ).withRetry(e -> e instanceof GatewayTimeoutException),

            // Step 4: Send notification (non-critical — compensation is just logging)
            SagaStep.of("send-notification",
                ctx -> {
                    var req = ctx.<PaymentRequest>get("request");
                    notifications.sendPaymentConfirmation(req.customerId(), ctx.get("transactionId"));
                    return "sent";
                },
                ctx -> log.warn("Notification compensation: payment failed after notification sent")
            )
        );

        return orchestrator.execute("payment-processing", context, steps);
    }
}
```

---

## 💃 Pattern 3: Saga — Choreography

> **When:** Simpler flows where each service reacts to events independently. No central coordinator.
> **Trade-off:** More autonomous but harder to track, debug, and ensure all steps complete.

### Choreography Flow

```
Payment Service          Account Service         Fraud Service          Gateway Service
     │                        │                       │                       │
     │  PaymentInitiated      │                       │                       │
     ├───────────────────────►│                       │                       │
     │                        │  FundsReserved        │                       │
     │                        ├──────────────────────►│                       │
     │                        │                       │  FraudCleared         │
     │                        │                       ├──────────────────────►│
     │                        │                       │                       │ PaymentCaptured
     │◄───────────────────────┼───────────────────────┼───────────────────────┤
     │                        │                       │                       │
     │  Update status         │                       │                       │
     │  Publish completion    │                       │                       │

ON FAILURE (FraudRejected):
     │                        │                       │                       │
     │                        │  ◄── FraudRejected ───┤                       │
     │                        │  Release reservation  │                       │
     │  ◄─ FundsReleased ────┤                       │                       │
     │  Update status: FAILED │                       │                       │
```

### Choreography Implementation

```java
// ── Each service listens and reacts independently ──

// Account Service
@Component
public class AccountEventHandler {

    @KafkaListener(topics = "payments.transaction.initiated")
    public void onPaymentInitiated(PaymentEvent.Initiated event, Acknowledgment ack) {
        try {
            var reservation = accountService.reserveFunds(event.customerId(), event.amount());
            eventPublisher.publish("accounts.funds.reserved",
                new FundsReserved(event.paymentId(), reservation.id(), event.amount()));
            ack.acknowledge();
        } catch (InsufficientFundsException e) {
            eventPublisher.publish("accounts.funds.reservation-failed",
                new FundsReservationFailed(event.paymentId(), e.getMessage()));
            ack.acknowledge();
        }
    }

    // Compensation: react to downstream failures
    @KafkaListener(topics = "fraud.check.rejected")
    public void onFraudRejected(FraudRejected event, Acknowledgment ack) {
        accountService.releaseReservation(event.paymentId());
        eventPublisher.publish("accounts.funds.released",
            new FundsReleased(event.paymentId()));
        ack.acknowledge();
    }
}

// Fraud Service
@Component
public class FraudEventHandler {

    @KafkaListener(topics = "accounts.funds.reserved")
    public void onFundsReserved(FundsReserved event, Acknowledgment ack) {
        var result = fraudEngine.evaluate(event.paymentId());
        if (result.approved()) {
            eventPublisher.publish("fraud.check.cleared",
                new FraudCleared(event.paymentId(), result.checkId()));
        } else {
            eventPublisher.publish("fraud.check.rejected",
                new FraudRejected(event.paymentId(), result.reason()));
        }
        ack.acknowledge();
    }
}
```

### Orchestration vs Choreography Decision Guide

| Factor | Orchestration | Choreography |
|--------|--------------|--------------|
| **Number of steps** | 3+ steps | 2-3 steps |
| **Compensation complexity** | Complex, ordered | Simple, independent |
| **Visibility** | Central — easy to monitor | Distributed — hard to trace |
| **Coupling** | Services coupled to orchestrator | Services coupled to events |
| **Failure handling** | Centralized, deterministic | Distributed, eventual |
| **Debugging** | Single place to look | Correlate across services |
| **Best for** | Payment processing, order fulfillment | Notifications, analytics, simple reactions |

---

## 📤 Pattern 4: Transactional Outbox

> **Problem:** How to atomically update your database AND publish an event?
> **Guaranteed solution:** Write event to an outbox table in the SAME database transaction, then relay to Kafka.

### The Dual-Write Problem

```
❌ THE PROBLEM — Dual write is NOT atomic:

  1. Save to MongoDB     ✅ succeeds
  2. Publish to Kafka    ❌ fails (network blip)
  
  Result: Database updated but event never published — inconsistency!

  OR:

  1. Publish to Kafka    ✅ succeeds
  2. Save to MongoDB     ❌ fails (constraint violation)
  
  Result: Event published but database not updated — inconsistency!


✅ THE SOLUTION — Outbox pattern:

  1. Save to MongoDB  }
  2. Save to outbox   } ← SAME MongoDB transaction — atomic
  
  3. Poller reads outbox → publishes to Kafka (separate process)
  4. Mark outbox entry as published
  
  If step 3 fails, poller retries. Message published AT LEAST ONCE.
  Consumer must be idempotent.
```

### Outbox Implementation — MongoDB

```java
// ── Outbox document ──
@Document("outbox_events")
@CompoundIndex(name = "status_created_idx", def = "{'status': 1, 'createdAt': 1}")
public record OutboxEvent(
    @Id String id,
    String aggregateId,          // Kafka key
    String aggregateType,        // "Payment"
    String eventType,            // "payment.captured"
    String topic,                // "payments.transaction.captured"
    String payload,              // Serialized event JSON
    Map<String, String> headers, // Kafka headers (correlationId, causationId)
    OutboxStatus status,         // PENDING, PUBLISHED, FAILED
    Instant createdAt,
    Instant publishedAt,
    int attemptCount,
    String lastError
) {
    public enum OutboxStatus { PENDING, PUBLISHED, FAILED }
}

// ── Business service — writes to outbox in same transaction ──
@Component
public class PaymentService {

    private final MongoTemplate mongoTemplate;
    private final ObjectMapper mapper;
    private final TransactionTemplate transactionTemplate;

    public Payment capturePayment(String paymentId, Money settledAmount) {
        return transactionTemplate.execute(status -> {
            // Step 1: Update payment state
            var payment = mongoTemplate.findById(paymentId, Payment.class);
            var updated = payment.withStatus(PaymentStatus.CAPTURED)
                                 .withSettledAmount(settledAmount);
            mongoTemplate.save(updated);

            // Step 2: Write event to outbox — SAME transaction
            var event = new PaymentEvent.Captured(
                paymentId, Instant.now(), settledAmount, UUID.randomUUID().toString());
            var outbox = new OutboxEvent(
                UUID.randomUUID().toString(),
                paymentId,
                "Payment",
                "payment.captured",
                "payments.transaction.captured",
                mapper.writeValueAsString(event),
                Map.of(
                    "correlationId", MDC.get("correlationId"),
                    "eventId", event.paymentId()
                ),
                OutboxStatus.PENDING,
                Instant.now(),
                null, 0, null
            );
            mongoTemplate.insert(outbox);

            return updated;
        });
    }
}

// ── Outbox poller — separate thread, at-least-once delivery ──
@Component
public class OutboxPoller {

    private final MongoTemplate mongoTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final MeterRegistry meterRegistry;

    @Scheduled(fixedDelayString = "${outbox.poll-interval:100}")
    public void pollAndPublish() {
        var pending = mongoTemplate.find(
            Query.query(Criteria.where("status").is("PENDING")
                    .and("attemptCount").lt(5))
                .limit(100)
                .with(Sort.by("createdAt")),
            OutboxEvent.class
        );

        if (pending.isEmpty()) return;

        for (var event : pending) {
            try {
                var record = new ProducerRecord<String, String>(
                    event.topic(), event.aggregateId(), event.payload());

                // Copy headers
                event.headers().forEach((k, v) ->
                    record.headers().add(k, v.getBytes(StandardCharsets.UTF_8)));

                kafkaTemplate.send(record).get(5, TimeUnit.SECONDS);

                // Mark as published
                mongoTemplate.updateFirst(
                    Query.query(Criteria.where("id").is(event.id())),
                    Update.update("status", "PUBLISHED")
                          .set("publishedAt", Instant.now()),
                    OutboxEvent.class
                );

                meterRegistry.counter("outbox.published", "topic", event.topic()).increment();

            } catch (Exception e) {
                log.warn("Outbox publish failed for {}: {}", event.id(), e.getMessage());
                mongoTemplate.updateFirst(
                    Query.query(Criteria.where("id").is(event.id())),
                    Update.update("attemptCount", event.attemptCount() + 1)
                          .set("lastError", e.getMessage()),
                    OutboxEvent.class
                );
                meterRegistry.counter("outbox.failure", "topic", event.topic()).increment();
            }
        }
    }

    // Cleanup — remove published events older than retention period
    @Scheduled(cron = "0 0 * * * *")  // Every hour
    public void cleanup() {
        var cutoff = Instant.now().minus(Duration.ofDays(7));
        var result = mongoTemplate.remove(
            Query.query(Criteria.where("status").is("PUBLISHED")
                    .and("publishedAt").lt(cutoff)),
            OutboxEvent.class
        );
        log.info("Outbox cleanup: removed {} published events", result.getDeletedCount());
    }
}
```

### Outbox Monitoring

```java
// Alert when outbox falls behind
@Component
public class OutboxHealthIndicator implements HealthIndicator {

    private final MongoTemplate mongoTemplate;

    @Override
    public Health health() {
        long pendingCount = mongoTemplate.count(
            Query.query(Criteria.where("status").is("PENDING")),
            OutboxEvent.class
        );

        long failedCount = mongoTemplate.count(
            Query.query(Criteria.where("status").is("PENDING")
                    .and("attemptCount").gte(3)),
            OutboxEvent.class
        );

        var builder = pendingCount < 1000 ? Health.up() : Health.down();
        return builder
            .withDetail("pendingEvents", pendingCount)
            .withDetail("failingEvents", failedCount)
            .build();
    }
}
```

---

## 🔐 Pattern 5: Idempotent Consumer

> **Rule:** In event-driven systems, EVERY consumer MUST be idempotent. Messages WILL be delivered more than once.

### Why Duplicates Happen

```
1. Producer retry     → Broker received but ack was lost → producer resends
2. Consumer rebalance → Offset not committed → message redelivered to new consumer
3. Outbox retry       → Published but poller didn't mark as published → resent
4. DLT replay         → Ops team replays dead-letter messages
5. Network partition   → Temporary duplication during partition healing
```

### Idempotency Strategies

```
Strategy 1: NATURAL IDEMPOTENCY
  The operation is inherently idempotent.
  "Set balance to $100" — doing it twice = same result
  "Upsert record with id X" — doing it twice = same result
  ✅ Best case — design operations to be naturally idempotent

Strategy 2: IDEMPOTENCY KEY (Dedup Table)
  Store processed event IDs, skip duplicates.
  "If eventId already processed, skip"
  ✅ Most common approach

Strategy 3: CONDITIONAL UPDATE (Optimistic Locking)
  "Update where version = N" — second attempt sees version N+1, fails gracefully
  ✅ Good for state transitions

Strategy 4: EVENT SEQUENCE (Ordering Guard)
  "Process only if event.sequenceNumber > lastProcessed"
  ✅ Good for ordered streams
```

### Idempotency Key — Production Implementation

```java
@Component
public class IdempotentEventProcessor {

    private final MongoTemplate mongoTemplate;

    /**
     * Process an event exactly once. Returns true if processed, false if duplicate.
     */
    public boolean processOnce(String eventId, Runnable action) {
        // Atomic insert — fails with DuplicateKeyException if already exists
        try {
            mongoTemplate.insert(new ProcessedEvent(
                eventId, ProcessingStatus.IN_PROGRESS, Instant.now(), null));
        } catch (DuplicateKeyException e) {
            // Check if it completed or is still in progress
            var existing = mongoTemplate.findById(eventId, ProcessedEvent.class);
            if (existing != null && existing.status() == ProcessingStatus.COMPLETED) {
                log.info("Duplicate event skipped: {}", eventId);
                return false;
            }
            // If IN_PROGRESS, another instance might be processing
            // Check if it's been stuck (> 5 min = treat as abandoned)
            if (existing != null && existing.startedAt()
                    .isBefore(Instant.now().minus(Duration.ofMinutes(5)))) {
                log.warn("Abandoned event detected, reprocessing: {}", eventId);
                mongoTemplate.remove(Query.query(Criteria.where("_id").is(eventId)),
                    ProcessedEvent.class);
                return processOnce(eventId, action); // Recursive retry
            }
            return false;
        }

        try {
            action.run();
            mongoTemplate.updateFirst(
                Query.query(Criteria.where("_id").is(eventId)),
                Update.update("status", "COMPLETED")
                      .set("completedAt", Instant.now()),
                ProcessedEvent.class
            );
            return true;
        } catch (Exception e) {
            // Remove marker so retry can re-attempt
            mongoTemplate.remove(
                Query.query(Criteria.where("_id").is(eventId)),
                ProcessedEvent.class
            );
            throw e;
        }
    }
}

// Usage in consumer
@KafkaListener(topics = "payments.transaction.captured")
public void onPaymentCaptured(
        @Payload PaymentEvent.Captured event,
        @Header("eventId") String eventId,
        Acknowledgment ack) {

    boolean processed = idempotentProcessor.processOnce(eventId,
        () -> settlementService.settle(event));

    if (!processed) {
        log.debug("Event {} already processed, acknowledging", eventId);
    }
    ack.acknowledge();
}
```

---

## 📊 Pattern 6: CQRS (Command Query Responsibility Segregation)

> **When:** Read and write models have fundamentally different shapes, performance needs, or scaling requirements.

### CQRS Architecture

```
                    WRITE SIDE                          READ SIDE
                    (Commands)                          (Queries)
                         │                                  ▲
                         ▼                                  │
              ┌─────────────────┐                ┌─────────────────┐
              │  Command Handler │                │  Query Handler   │
              └────────┬────────┘                └────────┬────────┘
                       │                                  │
                       ▼                                  │
              ┌─────────────────┐                ┌─────────────────┐
              │  Write Model     │                │  Read Model      │
              │  (Domain Logic)  │                │  (Projections)   │
              └────────┬────────┘                └────────▲────────┘
                       │                                  │
                       ▼                                  │
              ┌─────────────────┐    Events      ┌─────────────────┐
              │  Write Database  │──────────────►│  Read Database   │
              │  (MongoDB)       │   (Kafka)      │  (Elasticsearch, │
              └─────────────────┘                │   Redis, etc.)   │
                                                  └─────────────────┘
```

### CQRS Implementation

```java
// ── WRITE SIDE — Command handling ──
public sealed interface PaymentCommand {
    record InitiatePayment(String customerId, Money amount, PaymentMethod method,
                           String idempotencyKey) implements PaymentCommand {}
    record CapturePayment(String paymentId) implements PaymentCommand {}
    record RefundPayment(String paymentId, Money amount, String reason) implements PaymentCommand {}
}

@Component
public class PaymentCommandHandler {

    private final PaymentRepository repository;  // Write-optimized (MongoDB)
    private final EventPublisher eventPublisher;

    public PaymentResult handle(PaymentCommand command) {
        return switch (command) {
            case InitiatePayment cmd -> {
                var payment = Payment.initiate(cmd.customerId(), cmd.amount(), cmd.method());
                repository.save(payment);
                eventPublisher.publish(payment.domainEvents());
                yield PaymentResult.success(payment.id());
            }
            case CapturePayment cmd -> {
                var payment = repository.findById(cmd.paymentId())
                    .orElseThrow(() -> new PaymentNotFoundException(cmd.paymentId()));
                payment.capture();
                repository.save(payment);
                eventPublisher.publish(payment.domainEvents());
                yield PaymentResult.success(payment.id());
            }
            case RefundPayment cmd -> {
                var payment = repository.findById(cmd.paymentId())
                    .orElseThrow(() -> new PaymentNotFoundException(cmd.paymentId()));
                payment.refund(cmd.amount(), cmd.reason());
                repository.save(payment);
                eventPublisher.publish(payment.domainEvents());
                yield PaymentResult.success(payment.id());
            }
        };
    }
}

// ── READ SIDE — Query handling with projections ──
@Component
public class PaymentQueryHandler {

    private final PaymentReadRepository readRepository;  // Read-optimized

    public PaymentSummary getPaymentSummary(String paymentId) {
        return readRepository.findSummary(paymentId)
            .orElseThrow(() -> new PaymentNotFoundException(paymentId));
    }

    public Page<PaymentListItem> searchPayments(PaymentSearchCriteria criteria, Pageable pageable) {
        return readRepository.search(criteria, pageable);
    }

    public DashboardStats getDashboardStats(DateRange range) {
        return readRepository.computeStats(range);
    }
}

// ── Projection builder — listens to events, builds read models ──
@Component
public class PaymentProjectionBuilder {

    private final PaymentReadRepository readRepository;

    @KafkaListener(topics = "payments.transaction.#", groupId = "payment-projection-builder")
    public void onPaymentEvent(EventEnvelope<PaymentEvent> envelope, Acknowledgment ack) {
        var event = envelope.payload();

        switch (event) {
            case PaymentEvent.Initiated e -> readRepository.upsert(
                new PaymentReadModel(e.paymentId(), e.customerId(), e.amount(),
                    "INITIATED", e.occurredAt(), null, null));

            case PaymentEvent.Captured e -> readRepository.updateFields(e.paymentId(),
                Map.of("status", "CAPTURED",
                       "settledAmount", e.settledAmount(),
                       "capturedAt", e.occurredAt()));

            case PaymentEvent.Failed e -> readRepository.updateFields(e.paymentId(),
                Map.of("status", "FAILED",
                       "failureReason", e.reason(),
                       "failedAt", e.occurredAt()));

            case PaymentEvent.Refunded e -> readRepository.updateFields(e.paymentId(),
                Map.of("status", "REFUNDED",
                       "refundAmount", e.refundAmount(),
                       "refundedAt", e.occurredAt()));

            default -> log.debug("Unhandled event type for projection: {}",
                event.getClass().getSimpleName());
        }

        ack.acknowledge();
    }
}
```

### When CQRS Is Worth It

```
✅ USE CQRS when:
  • Read and write models are very different shapes
  • Read side needs different DB (Elasticsearch for search, Redis for dashboards)
  • Read load vastly exceeds write load (scale independently)
  • Complex queries that are expensive on the write model
  • Multiple read projections needed for different consumers

❌ DON'T USE CQRS when:
  • Read and write models are the same shape (just use one model)
  • Low data volume / simple CRUD
  • Strong consistency is required (CQRS is eventually consistent)
  • Team isn't comfortable with eventual consistency debugging
```

---

## 📸 Pattern 7: Event Sourcing

> **When:** You need complete audit trail, temporal queries, and the ability to reconstruct state at any point in time.
> **Core idea:** Store events, not state. Current state = replay all events.

### Event Sourcing vs State-Based

```
STATE-BASED (Traditional):
  Payment { id: "pay-1", status: "CAPTURED", amount: 100 }
  ← Only the CURRENT state is known. History is lost.

EVENT-SOURCED:
  1. PaymentInitiated { paymentId: "pay-1", amount: 100, customerId: "c1" }
  2. PaymentValidated  { paymentId: "pay-1", result: "PASS" }
  3. PaymentAuthorized { paymentId: "pay-1", authCode: "AUTH-1" }
  4. PaymentCaptured   { paymentId: "pay-1", settledAmount: 100 }
  ← Full history preserved. Can reconstruct state at any point.
```

### Event-Sourced Aggregate

```java
public class PaymentAggregate {

    private String paymentId;
    private Money amount;
    private PaymentStatus status;
    private String customerId;
    private String authCode;
    private Money settledAmount;
    private long version;

    private final List<PaymentEvent> uncommittedEvents = new ArrayList<>();

    // ── Reconstruct from events ──
    public static PaymentAggregate fromHistory(List<PaymentEvent> events) {
        var aggregate = new PaymentAggregate();
        events.forEach(aggregate::apply);
        aggregate.uncommittedEvents.clear();  // History events are already committed
        return aggregate;
    }

    // ── Command methods — validate + emit events ──
    public void initiate(String customerId, Money amount, PaymentMethod method) {
        if (status != null) throw new IllegalStateException("Already initiated");
        emit(new PaymentEvent.Initiated(
            UUID.randomUUID().toString(), Instant.now(), amount, customerId, method,
            UUID.randomUUID().toString()));
    }

    public void capture() {
        if (status != PaymentStatus.AUTHORIZED)
            throw new IllegalStateException("Cannot capture from status: " + status);
        emit(new PaymentEvent.Captured(paymentId, Instant.now(), amount, 
            UUID.randomUUID().toString(), Money.ZERO));
    }

    public void refund(Money refundAmount, String reason) {
        if (status != PaymentStatus.CAPTURED)
            throw new IllegalStateException("Cannot refund from status: " + status);
        if (refundAmount.compareTo(settledAmount) > 0)
            throw new IllegalArgumentException("Refund exceeds settled amount");
        emit(new PaymentEvent.Refunded(paymentId, Instant.now(), refundAmount,
            UUID.randomUUID().toString(), reason));
    }

    // ── Apply events — mutate state (no validation here) ──
    private void apply(PaymentEvent event) {
        switch (event) {
            case PaymentEvent.Initiated e -> {
                this.paymentId = e.paymentId();
                this.amount = e.amount();
                this.customerId = e.customerId();
                this.status = PaymentStatus.INITIATED;
            }
            case PaymentEvent.Authorized e -> {
                this.authCode = e.authorizationCode();
                this.status = PaymentStatus.AUTHORIZED;
            }
            case PaymentEvent.Captured e -> {
                this.settledAmount = e.settledAmount();
                this.status = PaymentStatus.CAPTURED;
            }
            case PaymentEvent.Failed e -> this.status = PaymentStatus.FAILED;
            case PaymentEvent.Refunded e -> this.status = PaymentStatus.REFUNDED;
            default -> { }
        }
        this.version++;
    }

    private void emit(PaymentEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }

    public List<PaymentEvent> uncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void markCommitted() { uncommittedEvents.clear(); }
}
```

### Event Store — MongoDB Implementation

```java
@Component
public class MongoEventStore {

    private final MongoTemplate mongoTemplate;

    // ── Append events with optimistic concurrency ──
    public void append(String aggregateId, long expectedVersion, List<PaymentEvent> events) {
        var documents = new ArrayList<EventDocument>();

        for (int i = 0; i < events.size(); i++) {
            documents.add(new EventDocument(
                UUID.randomUUID().toString(),
                aggregateId,
                expectedVersion + i + 1,   // Sequential version
                events.get(i).getClass().getSimpleName(),
                serialize(events.get(i)),
                Instant.now()
            ));
        }

        try {
            mongoTemplate.insertAll(documents);
        } catch (DuplicateKeyException e) {
            // Unique index on (aggregateId, version) prevents concurrent writes
            throw new OptimisticConcurrencyException(
                "Aggregate " + aggregateId + " was modified concurrently");
        }
    }

    // ── Load all events for an aggregate ──
    public List<PaymentEvent> loadEvents(String aggregateId) {
        return mongoTemplate.find(
            Query.query(Criteria.where("aggregateId").is(aggregateId))
                 .with(Sort.by("version")),
            EventDocument.class
        ).stream()
         .map(doc -> deserialize(doc.payload(), doc.eventType()))
         .toList();
    }

    // ── Load events after a specific version (for snapshots) ──
    public List<PaymentEvent> loadEventsAfter(String aggregateId, long afterVersion) {
        return mongoTemplate.find(
            Query.query(Criteria.where("aggregateId").is(aggregateId)
                    .and("version").gt(afterVersion))
                 .with(Sort.by("version")),
            EventDocument.class
        ).stream()
         .map(doc -> deserialize(doc.payload(), doc.eventType()))
         .toList();
    }
}

// Unique compound index ensures no two events for same aggregate have same version
@Document("event_store")
@CompoundIndex(name = "aggregate_version_idx",
    def = "{'aggregateId': 1, 'version': 1}", unique = true)
public record EventDocument(
    @Id String id,
    String aggregateId,
    long version,
    String eventType,
    String payload,
    Instant createdAt
) {}
```

### Snapshots — Performance Optimization

```java
@Component
public class SnapshotStore {

    private final MongoTemplate mongoTemplate;
    private static final int SNAPSHOT_INTERVAL = 50;  // Snapshot every 50 events

    public PaymentAggregate load(String aggregateId, MongoEventStore eventStore) {
        // Try loading from snapshot first
        var snapshot = mongoTemplate.findById(aggregateId, AggregateSnapshot.class);

        List<PaymentEvent> events;
        PaymentAggregate aggregate;

        if (snapshot != null) {
            // Replay only events AFTER the snapshot
            aggregate = deserialize(snapshot.state());
            events = eventStore.loadEventsAfter(aggregateId, snapshot.version());
        } else {
            // No snapshot — replay everything
            aggregate = new PaymentAggregate();
            events = eventStore.loadEvents(aggregateId);
        }

        events.forEach(aggregate::apply);

        // Create snapshot if we've replayed enough events
        long currentVersion = (snapshot != null ? snapshot.version() : 0) + events.size();
        if (events.size() >= SNAPSHOT_INTERVAL) {
            mongoTemplate.save(new AggregateSnapshot(
                aggregateId, currentVersion, serialize(aggregate), Instant.now()));
        }

        return aggregate;
    }
}
```

---

## 📐 Pattern 8: Schema Evolution

> **Rule:** Events are a contract. Treat schema changes like API versioning.

### Evolution Strategies

```
STRATEGY 1: ADDITIVE ONLY (Recommended)
  ✅ Add optional fields with defaults
  ✅ Add new event types
  ❌ Never remove fields
  ❌ Never rename fields
  ❌ Never change field types
  
  v1: { paymentId, amount, customerId }
  v2: { paymentId, amount, customerId, riskScore? }     ← optional new field
  
  Consumers ignore unknown fields → forward compatible
  Missing optional fields get defaults → backward compatible


STRATEGY 2: VERSIONED EVENTS
  Maintain multiple versions simultaneously
  
  Topic: payments.transaction.initiated
  Messages contain: { schemaVersion: 1, ... } or { schemaVersion: 2, ... }
  
  Consumer handles both:
    if (schemaVersion == 1) → handle v1
    if (schemaVersion == 2) → handle v2


STRATEGY 3: UPCASTING (Event Sourcing)
  Transform old events to current schema during replay
  
  v1 stored: { amount: "100.00" }           ← String
  Upcaster:  { amount: { value: "100.00", currency: "USD" } }  ← Money object
  
  Stored format never changes. Upcaster applied at read time.
```

### Upcaster Implementation

```java
public interface EventUpcaster {
    boolean canUpcast(String eventType, int fromVersion);
    JsonNode upcast(JsonNode eventData, int fromVersion);
}

@Component
public class PaymentInitiatedUpcaster implements EventUpcaster {

    @Override
    public boolean canUpcast(String eventType, int fromVersion) {
        return "PaymentInitiated".equals(eventType) && fromVersion < 2;
    }

    @Override
    public JsonNode upcast(JsonNode eventData, int fromVersion) {
        if (fromVersion == 1) {
            // v1 had amount as string, v2 has amount as Money object
            var node = (ObjectNode) eventData;
            String amountStr = node.get("amount").asText();
            node.remove("amount");
            var amountNode = node.putObject("amount");
            amountNode.put("value", amountStr);
            amountNode.put("currency", "USD");  // Default for v1 events
        }
        return eventData;
    }
}

@Component
public class UpcasterChain {
    private final List<EventUpcaster> upcasters;

    public JsonNode upcast(String eventType, int storedVersion, int targetVersion, JsonNode data) {
        var current = data;
        for (int v = storedVersion; v < targetVersion; v++) {
            for (var upcaster : upcasters) {
                if (upcaster.canUpcast(eventType, v)) {
                    current = upcaster.upcast(current, v);
                }
            }
        }
        return current;
    }
}
```

---

## 📊 Pattern 9: Event-Driven Observability

### Correlation & Causation Tracking

```java
@Component
public class EventCorrelationFilter implements KafkaListenerInterceptor {

    public void onConsume(ConsumerRecord<String, ?> record) {
        var headers = record.headers();

        // Extract or generate correlation ID
        String correlationId = headerValue(headers, "correlationId")
            .orElse(UUID.randomUUID().toString());

        // Event that caused this processing
        String causationId = headerValue(headers, "eventId")
            .orElse("unknown");

        // Set in MDC for structured logging
        MDC.put("correlationId", correlationId);
        MDC.put("causationId", causationId);
        MDC.put("eventId", UUID.randomUUID().toString());

        log.info("Processing event: topic={}, key={}, correlationId={}",
            record.topic(), record.key(), correlationId);
    }

    // Propagate to outgoing events
    public void onProduce(ProducerRecord<String, ?> record) {
        record.headers().add("correlationId", MDC.get("correlationId").getBytes());
        record.headers().add("causationId", MDC.get("eventId").getBytes());
        record.headers().add("eventId", UUID.randomUUID().toString().getBytes());
    }
}
```

### Event Flow Visualization (Structured Logging)

```json
// Log output for tracing an entire payment flow across services
{"timestamp":"2026-02-07T10:00:01Z","service":"payment-service","correlationId":"corr-abc","causationId":"cmd-1","eventId":"evt-1","message":"PaymentInitiated","paymentId":"pay-123"}
{"timestamp":"2026-02-07T10:00:02Z","service":"account-service","correlationId":"corr-abc","causationId":"evt-1","eventId":"evt-2","message":"FundsReserved","paymentId":"pay-123"}
{"timestamp":"2026-02-07T10:00:03Z","service":"fraud-service","correlationId":"corr-abc","causationId":"evt-2","eventId":"evt-3","message":"FraudCleared","paymentId":"pay-123"}
{"timestamp":"2026-02-07T10:00:05Z","service":"gateway-service","correlationId":"corr-abc","causationId":"evt-3","eventId":"evt-4","message":"PaymentCaptured","paymentId":"pay-123"}
```

### Key Metrics for Event-Driven Systems

```java
@Component
public class EventDrivenMetrics {

    private final MeterRegistry registry;

    // ── Publishing lag ──
    public void recordPublishLatency(String topic, Duration latency) {
        registry.timer("events.publish.latency", "topic", topic).record(latency);
    }

    // ── Processing lag (event time → processing time) ──
    public void recordProcessingLag(String topic, Instant eventTime) {
        Duration lag = Duration.between(eventTime, Instant.now());
        registry.timer("events.processing.lag", "topic", topic).record(lag);
    }

    // ── End-to-end saga duration ──
    public void recordSagaDuration(String sagaName, Duration duration, String outcome) {
        registry.timer("saga.duration",
            "saga", sagaName, "outcome", outcome).record(duration);
    }

    // ── Event throughput ──
    public void recordEventProcessed(String topic, String eventType) {
        registry.counter("events.processed",
            "topic", topic, "type", eventType).increment();
    }

    // ── Projection lag (events behind) ──
    public void recordProjectionLag(String projection, long eventsBehind) {
        registry.gauge("projection.lag",
            Tags.of("projection", projection), eventsBehind);
    }
}
```

### Alert Rules

```yaml
# Event processing lag > 5 minutes
- alert: HighEventProcessingLag
  expr: histogram_quantile(0.99, events_processing_lag_seconds_bucket) > 300
  for: 5m
  severity: warning

# Outbox backing up
- alert: OutboxBacklog
  expr: outbox_pending_count > 500
  for: 2m
  severity: critical

# Saga failure rate > 5%
- alert: SagaFailureRate
  expr: rate(saga_duration_seconds_count{outcome="failed"}[5m]) /
        rate(saga_duration_seconds_count[5m]) > 0.05
  for: 5m
  severity: critical

# Projection falling behind
- alert: ProjectionLagHigh
  expr: projection_lag > 10000
  for: 10m
  severity: warning

# Dead letter accumulation
- alert: DeadLetterAccumulating
  expr: increase(outbox_failure_total[1h]) > 50
  for: 0m
  severity: critical
```

---

## 🚫 EDA Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Dual write (DB + Kafka)** | Not atomic — data divergence guaranteed | Outbox pattern or CDC |
| **Synchronous event publishing** | Defeats the purpose of async; blocks caller | Async publish, outbox for durability |
| **Fat events with secrets** | PII/credentials leak across services | Strip sensitive data; use reference IDs |
| **No idempotency** | Duplicate processing = duplicate payments | Every consumer must be idempotent |
| **God event** | One event type carries everything | Split into domain-specific events |
| **Event as command** | "PleaseProcessPayment" published as event | Use commands for directed requests |
| **Temporal coupling** | Consumer assumes events arrive within X ms | Design for arbitrary delays |
| **Missing correlation ID** | Cannot trace flow across services | Propagate correlation in every event |
| **Ignoring schema evolution** | Breaking change crashes all consumers | Additive-only or versioned schemas |
| **No dead-letter handling** | Failed events disappear silently | DLT + monitoring + replay capability |
| **Circular event chains** | A triggers B triggers A triggers B... | Detect cycles via causation chain |
| **Event sourcing everything** | Massive complexity for simple CRUD | Use event sourcing only where needed |
| **Saga without persistence** | Orchestrator crash = orphaned steps | Persist saga state for recovery |
| **No consumer lag monitoring** | System falls behind without anyone knowing | Monitor and alert on lag |

---

## ⚡ Quick Reference: Which Pattern When?

| Scenario | Primary Pattern | Supporting Patterns |
|---------|----------------|---------------------|
| Update DB + publish event atomically | **Outbox** | Idempotent consumer |
| Multi-service transaction (complex) | **Saga — Orchestration** | Outbox, Idempotency |
| Multi-service reaction (simple) | **Saga — Choreography** | Correlation ID, Idempotency |
| Read/write model mismatch | **CQRS** | Event-carried state, Projections |
| Full audit trail & temporal queries | **Event Sourcing** | Snapshots, Upcasting |
| Prevent duplicate processing | **Idempotent Consumer** | Dedup table, Conditional update |
| Decouple unknown consumers | **Event Notification** | Pub-sub, Schema registry |
| Consumer needs all data | **Event-Carried State** | Fat events, Schema evolution |
| Debug cross-service flows | **Correlation/Causation** | Structured logging, Tracing |
| Schema changes over time | **Additive Evolution** | Upcasting, Versioning |

---

## 💡 Golden Rules of Event-Driven Architecture

```
1.  Events are FACTS — they represent things that HAPPENED. They cannot be rejected.
2.  Commands are REQUESTS — they represent things you WANT to happen. They can fail.
3.  EVERY consumer MUST be idempotent — duplicates are a certainty, not a possibility.
4.  NEVER dual-write — use the outbox pattern for atomic DB + event publishing.
5.  Correlation IDs are mandatory — without them, debugging distributed flows is impossible.
6.  Schema evolution is a first-class concern — treat events like versioned APIs.
7.  Eventual consistency is a feature, not a bug — but define your SLAs clearly.
8.  Event sourcing is powerful but expensive — use it where audit & history justify the cost.
9.  Monitor the GAPS — consumer lag, outbox backlog, and DLT depth are your early warnings.
10. Design for failure at every step — compensate, retry, dead-letter, alert, and recover.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Kafka / MongoDB / Artemis / Resilience4j*
