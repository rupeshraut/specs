# 🔥 Apache Kafka Patterns Cheat Sheet

> **Purpose:** Production-ready Kafka patterns for enterprise event-driven systems. Reference before designing any producer, consumer, topology, or retry strategy.
> **Stack context:** Java 21+ / Spring Boot 3.x / Spring Kafka / Confluent / MongoDB / Resilience4j

---

## 📋 The Kafka Design Decision Framework

Before writing any Kafka code, answer:

| Question | Determines |
|----------|-----------|
| What's the **delivery guarantee** needed? | at-most-once / at-least-once / exactly-once |
| Is the consumer **idempotent**? | Safe to reprocess or not |
| Does **ordering** matter? | Partitioning strategy |
| What's the **message size**? | Compression, chunking, claim-check |
| What's the **retention** requirement? | Topic configuration |
| What's the **throughput** vs **latency** trade-off? | Batching, linger, acks |
| What happens when a message **can't be processed**? | Retry topology, DLT |
| Is this **event notification** or **event-carried state transfer**? | Payload design |
| Who are the **consumers** (known or unknown)? | Topic naming, schema evolution |
| What's the **replay** requirement? | Compaction, retention, consumer reset |

---

## 🏗️ Pattern 1: Topic Design & Naming

### Topic Naming Convention

```
<domain>.<entity>.<event-type>

Examples:
  payments.transaction.initiated
  payments.transaction.completed
  payments.transaction.failed
  accounts.balance.updated
  fraud.check.requested
  fraud.check.completed
  notifications.email.requested

Retry & DLT:
  payments.transaction.initiated.retry-1
  payments.transaction.initiated.retry-2
  payments.transaction.initiated.dlt

Internal/Compacted:
  payments.transaction.state          (compacted — latest state per key)
  accounts.balance.snapshot           (compacted — current balances)
```

### Topic Configuration Decision Guide

| Use Case | Partitions | Retention | Cleanup Policy | Replication |
|----------|-----------|-----------|----------------|-------------|
| **High-throughput events** (clicks, logs) | 12-30 | 7 days | delete | 3 |
| **Business events** (payments, orders) | 6-12 | 30 days | delete | 3 |
| **State/snapshot** (account balances) | 6-12 | ∞ | compact | 3 |
| **Retry topics** | Same as source | 7 days | delete | 3 |
| **Dead-letter topics** | 1-3 | ∞ (or 90 days) | delete | 3 |
| **Audit/compliance** | 6-12 | ∞ (or years) | delete | 3 |
| **Commands** (request-reply) | 6-12 | 1 day | delete | 3 |

### Topic Creation (Production Settings)

```java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic paymentInitiatedTopic() {
        return TopicBuilder.name("payments.transaction.initiated")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(Duration.ofDays(30).toMillis()))
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")       // ISR safety
            .config(TopicConfig.COMPRESSION_TYPE_CONFIG, "lz4")          // Best throughput/ratio
            .config(TopicConfig.MAX_MESSAGE_BYTES_CONFIG, "1048576")     // 1MB max
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, "delete")
            .build();
    }

    @Bean
    public NewTopic accountBalanceSnapshotTopic() {
        return TopicBuilder.name("accounts.balance.snapshot")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, "compact")
            .config(TopicConfig.MIN_COMPACTION_LAG_MS_CONFIG, String.valueOf(Duration.ofHours(1).toMillis()))
            .config(TopicConfig.DELETE_RETENTION_MS_CONFIG, String.valueOf(Duration.ofDays(1).toMillis()))
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
            .build();
    }

    @Bean
    public NewTopic deadLetterTopic() {
        return TopicBuilder.name("payments.transaction.initiated.dlt")
            .partitions(3)          // Low partition count — low volume expected
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, "-1")   // Infinite retention
            .build();
    }
}
```

---

## 🔑 Pattern 2: Partitioning Strategy

> **Rule:** Messages with the same key go to the same partition → guarantees ordering per key.

### Partitioning Decision Tree

```
Does ordering matter?
├── NO → Use round-robin (null key) for max throughput
└── YES → What's the ordering scope?
    ├── Per entity (per payment, per customer)
    │   → Key = entityId (e.g., paymentId, customerId)
    ├── Per aggregate (all events for an order)
    │   → Key = aggregateId (e.g., orderId)
    └── Global ordering (rare, avoid if possible)
        → Single partition (kills parallelism)
```

### Key Design Patterns

```java
// Simple entity key — ordering per payment
ProducerRecord<String, PaymentEvent> record = new ProducerRecord<>(
    "payments.transaction.initiated",
    payment.paymentId(),       // key — all events for this payment go to same partition
    paymentEvent
);

// Composite key — ordering per customer within a region
public record PaymentKey(String customerId, String region) {}
// All of customer-123's payments in US_EAST go to the same partition

// Custom partitioner — region-based routing
public class RegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int partitionCount = cluster.partitionCountForTopic(topic);
        if (key instanceof PaymentKey pk) {
            // Hash on region first, then spread within region
            return Math.abs(pk.region().hashCode()) % partitionCount;
        }
        return Math.abs(Utils.murmur2(keyBytes)) % partitionCount;
    }
}
```

### Partition Count Sizing

```
Formula:
  partitions = max(T/Pp, T/Cp)

Where:
  T  = target throughput (messages/sec)
  Pp = single producer throughput per partition
  Cp = single consumer throughput per partition

Example:
  Target: 10,000 msg/sec
  Producer per partition: 5,000 msg/sec
  Consumer per partition: 1,000 msg/sec
  Partitions = max(10000/5000, 10000/1000) = max(2, 10) = 10 → use 12 (round up)

Rules of thumb:
  • Start with 6-12 for most business topics
  • Never exceed number of consumers in the largest consumer group
  • More partitions = more parallelism but more broker overhead
  • Partition count can increase (rebalance), but NEVER decrease
  • Increasing partitions BREAKS key-based ordering for existing data
```

---

## 📤 Pattern 3: Producer Patterns

### Delivery Guarantee Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public Map<String, Object> producerConfigs() {
        return Map.ofEntries(
            // ── Serialization ──
            entry(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class),
            entry(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class),

            // ── Durability (ZERO message loss) ──
            entry(ProducerConfig.ACKS_CONFIG, "all"),          // Wait for ALL ISR replicas
            entry(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true),  // Dedup at broker level
            entry(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5), // Safe with idempotence

            // ── Retries ──
            entry(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE), // Retry forever (bounded by delivery.timeout.ms)
            entry(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000), // 2 min total delivery budget
            entry(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000),   // 30s per request
            entry(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100),       // 100ms between retries

            // ── Batching & Throughput ──
            entry(ProducerConfig.BATCH_SIZE_CONFIG, 32768),       // 32KB batch
            entry(ProducerConfig.LINGER_MS_CONFIG, 20),            // Wait 20ms to fill batch
            entry(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4"),  // Best throughput/ratio
            entry(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864)   // 64MB buffer
        );
    }
}
```

### Acks Decision Matrix

| acks | Durability | Latency | Throughput | Use Case |
|------|-----------|---------|------------|----------|
| **0** | None — fire and forget | Lowest | Highest | Metrics, logs (loss OK) |
| **1** | Leader only — survives broker crash if leader lives | Medium | High | General events |
| **all (-1)** | Full ISR — survives any single broker failure | Highest | Lowest | Payments, financial data, audit |

### Producer Send Patterns

```java
@Component
public class PaymentEventPublisher {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    private final MeterRegistry meterRegistry;

    // ── Pattern A: Fire-and-forget (logs, metrics) ──
    public void publishMetric(String key, MetricEvent event) {
        kafkaTemplate.send("metrics.collected", key, event);
        // No callback — acceptable loss
    }

    // ── Pattern B: Async with callback (standard business events) ──
    public CompletableFuture<SendResult<String, PaymentEvent>> publishAsync(
            String topic, String key, PaymentEvent event) {

        return kafkaTemplate.send(topic, key, event)
            .thenApply(result -> {
                var metadata = result.getRecordMetadata();
                log.info("Published event: topic={}, partition={}, offset={}, key={}",
                    metadata.topic(), metadata.partition(), metadata.offset(), key);
                meterRegistry.counter("kafka.producer.success", "topic", topic).increment();
                return result;
            })
            .exceptionally(ex -> {
                log.error("Failed to publish event: topic={}, key={}", topic, key, ex);
                meterRegistry.counter("kafka.producer.failure", "topic", topic).increment();
                throw new EventPublishException("Failed to publish to " + topic, ex);
            });
    }

    // ── Pattern C: Sync with guaranteed delivery (payments, financial) ──
    public SendResult<String, PaymentEvent> publishSync(
            String topic, String key, PaymentEvent event) {
        try {
            return kafkaTemplate.send(topic, key, event)
                .get(10, TimeUnit.SECONDS);    // Block until broker confirms
        } catch (TimeoutException e) {
            throw new EventPublishException("Publish timed out after 10s", e);
        } catch (ExecutionException e) {
            throw new EventPublishException("Publish failed: " + e.getCause().getMessage(), e);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new EventPublishException("Publish interrupted", e);
        }
    }

    // ── Pattern D: Transactional (exactly-once with DB) ──
    public void publishTransactional(String key, PaymentEvent event) {
        kafkaTemplate.executeInTransaction(ops -> {
            ops.send("payments.transaction.initiated", key, event);
            ops.send("audit.payment.log", key, toAuditEvent(event));
            return true;
            // Both messages committed atomically, or neither
        });
    }
}
```

### Outbox Pattern (Transactional Messaging with MongoDB)

> **Problem:** How to atomically update MongoDB AND publish to Kafka?
> **Solution:** Write to an outbox collection in the same MongoDB transaction, then poll/CDC to Kafka.

```java
@Component
public class TransactionalOutbox {

    private final MongoTemplate mongoTemplate;

    @Transactional  // MongoDB transaction
    public void processPayment(Payment payment) {
        // Step 1: Business logic — update payment state
        payment = payment.withStatus(PaymentStatus.COMPLETED);
        mongoTemplate.save(payment);

        // Step 2: Write to outbox — same transaction
        var outboxEvent = new OutboxEvent(
            UUID.randomUUID().toString(),
            "payments.transaction.completed",
            payment.paymentId(),           // Kafka key
            serialize(payment),            // Kafka value
            Instant.now(),
            OutboxStatus.PENDING
        );
        mongoTemplate.insert(outboxEvent);
        // Both writes succeed or both fail — atomically
    }
}

// Outbox poller — runs on a schedule, publishes pending events to Kafka
@Component
public class OutboxPoller {

    @Scheduled(fixedDelay = 100)  // Poll every 100ms
    public void pollAndPublish() {
        var pending = mongoTemplate.find(
            Query.query(Criteria.where("status").is("PENDING"))
                 .limit(100)
                 .with(Sort.by("createdAt")),
            OutboxEvent.class
        );

        for (var event : pending) {
            try {
                kafkaTemplate.send(event.topic(), event.key(), event.payload())
                    .get(5, TimeUnit.SECONDS);

                mongoTemplate.updateFirst(
                    Query.query(Criteria.where("id").is(event.id())),
                    Update.update("status", "PUBLISHED")
                           .set("publishedAt", Instant.now()),
                    OutboxEvent.class
                );
            } catch (Exception e) {
                log.warn("Outbox publish failed for {}, will retry", event.id(), e);
                // Leave as PENDING — next poll will retry
            }
        }
    }
}
```

---

## 📥 Pattern 4: Consumer Patterns

### Consumer Configuration — Zero Message Loss

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public Map<String, Object> consumerConfigs() {
        return Map.ofEntries(
            // ── Serialization ──
            entry(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class),
            entry(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class),

            // ── Consumer Group ──
            entry(ConsumerConfig.GROUP_ID_CONFIG, "payment-processor-v1"),
            entry(ConsumerConfig.CLIENT_ID_CONFIG, "payment-processor-" + hostname()),

            // ── Offset Management ──
            entry(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false),    // MANUAL commits only
            entry(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"), // Don't skip messages

            // ── Polling ──
            entry(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100),        // Batch size per poll
            entry(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000), // 5 min max processing time
            entry(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1),           // Low latency
            entry(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500),       // Max wait for fetch.min.bytes

            // ── Session & Heartbeat ──
            entry(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000),    // 30s session timeout
            entry(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000), // 10s heartbeat

            // ── Isolation (for exactly-once) ──
            entry(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed")
        );
    }
}
```

### Spring Kafka Listener Patterns

```java
@Component
public class PaymentEventConsumer {

    // ── Pattern A: Simple single-record processing ──
    @KafkaListener(
        topics = "payments.transaction.initiated",
        groupId = "payment-processor-v1",
        concurrency = "3"          // 3 consumer threads
    )
    public void processPayment(
            @Payload PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Processing payment: key={}, partition={}, offset={}", key, partition, offset);

        try {
            paymentService.process(event);
            ack.acknowledge();  // Manual commit AFTER successful processing
        } catch (RetryableException e) {
            // Don't ack — Spring Kafka retry/error handler will manage this
            throw e;
        } catch (NonRetryableException e) {
            log.error("Non-retryable error, sending to DLT: key={}", key, e);
            ack.acknowledge();  // Ack to avoid blocking, error handler routes to DLT
            throw e;
        }
    }

    // ── Pattern B: Batch processing (high throughput) ──
    @KafkaListener(
        topics = "payments.settlement.batch",
        groupId = "settlement-processor-v1",
        containerFactory = "batchListenerFactory"
    )
    public void processBatch(
            List<ConsumerRecord<String, SettlementEvent>> records,
            Acknowledgment ack) {

        log.info("Received batch of {} records", records.size());

        var results = records.stream()
            .map(record -> processWithResult(record))
            .toList();

        var failures = results.stream().filter(Result::isFailure).toList();

        if (failures.isEmpty()) {
            ack.acknowledge();  // All succeeded — commit entire batch
        } else {
            // Route failures to DLT, ack the batch
            failures.forEach(f -> deadLetterPublisher.publish(f.record(), f.error()));
            ack.acknowledge();
        }
    }

    // ── Pattern C: Filter before processing ──
    @KafkaListener(
        topics = "payments.transaction.all",
        groupId = "high-value-processor",
        filter = "highValueFilter"
    )
    public void processHighValueOnly(@Payload PaymentEvent event, Acknowledgment ack) {
        // Only receives events that pass the filter
        enrichAndProcess(event);
        ack.acknowledge();
    }

    @Bean
    public RecordFilterStrategy<String, PaymentEvent> highValueFilter() {
        // Return TRUE to DISCARD (filter OUT), FALSE to KEEP
        return record -> record.value().amount().compareTo(new BigDecimal("10000")) < 0;
    }

    // ── Pattern D: Concurrent partition processing ──
    @KafkaListener(
        topics = "payments.transaction.initiated",
        groupId = "payment-processor-v2",
        concurrency = "${kafka.consumer.concurrency:6}"  // Match partition count
    )
    public void processWithConcurrency(@Payload PaymentEvent event, Acknowledgment ack) {
        // Each thread handles one or more partitions
        // Ordering guaranteed within each partition
        process(event);
        ack.acknowledge();
    }
}
```

### Batch Listener Factory

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> batchListenerFactory(
        ConsumerFactory<String, Object> consumerFactory) {

    var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
    factory.setConsumerFactory(consumerFactory);
    factory.setBatchListener(true);                         // Enable batch mode
    factory.setConcurrency(6);
    factory.getContainerProperties().setAckMode(AckMode.MANUAL);
    factory.getContainerProperties().setIdleBetweenPolls(100);
    return factory;
}
```

---

## 🔄 Pattern 5: Retry Topologies

### Non-Blocking Retry with Delay Topics

> **Problem:** Blocking retries hold up the partition, delaying all messages behind the failed one.
> **Solution:** Route failed messages to delay topics with increasing wait times.

```
Main Topic                    Retry-1 (1 min delay)    Retry-2 (5 min delay)    DLT
payments.process         →    payments.retry-1     →    payments.retry-2     →   payments.dlt
  │                             │                         │
  ├── Success → ack             ├── Success → ack         ├── Success → ack
  └── Failure → route           └── Failure → route       └── Failure → route to DLT
      to retry-1                    to retry-2                  + alert ops
```

### Spring Kafka Non-Blocking Retry

```java
@Configuration
public class KafkaRetryTopologyConfig {

    @Bean
    public RetryTopicConfiguration retryTopicConfig(KafkaTemplate<String, Object> template) {
        return RetryTopicConfigurationBuilder
            .newInstance()
            .exponentialBackoff(
                Duration.ofSeconds(30).toMillis(),      // Initial delay
                3.0,                                      // Multiplier
                Duration.ofMinutes(15).toMillis()          // Max delay
            )
            .maxRetryAttempts(4)                           // 30s → 90s → 270s → 810s → DLT
            .autoCreateTopics(true, 6, (short) 3)          // partitions, replication
            .retryTopicSuffix("-retry")
            .dltSuffix("-dlt")
            .setTopicSuffixingStrategy(TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE)
            .notRetryOn(
                ValidationException.class,
                JsonParseException.class,
                SerializationException.class,
                SecurityException.class
            )
            .includeTopics(List.of(
                "payments.transaction.initiated",
                "payments.settlement.requested"
            ))
            .dltHandlerMethod("deadLetterHandler", "handle")
            .create(template);
    }

    @Component("deadLetterHandler")
    public static class DeadLetterHandler {

        private final MongoTemplate mongoTemplate;
        private final MeterRegistry meterRegistry;

        public void handle(ConsumerRecord<String, Object> record,
                          @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage,
                          @Header(KafkaHeaders.ORIGINAL_TOPIC) String originalTopic,
                          @Header(KafkaHeaders.ORIGINAL_OFFSET) long originalOffset) {

            log.error("Message exhausted all retries: topic={}, key={}, error={}",
                originalTopic, record.key(), errorMessage);

            // Persist for investigation and manual replay
            mongoTemplate.insert(new DeadLetterRecord(
                record.key(),
                originalTopic,
                originalOffset,
                record.value(),
                errorMessage,
                Instant.now()
            ));

            meterRegistry.counter("kafka.dlt.received",
                "original_topic", originalTopic).increment();
        }
    }
}
```

### Custom Tiered Retry with Headers

```java
@Component
public class TieredRetryHandler {

    private static final List<RetryTier> TIERS = List.of(
        new RetryTier("retry-1", Duration.ofSeconds(30),  3),  // 3 attempts, 30s apart
        new RetryTier("retry-2", Duration.ofMinutes(5),   3),  // 3 attempts, 5min apart
        new RetryTier("retry-3", Duration.ofMinutes(30),  2)   // 2 attempts, 30min apart
    );

    public void handleFailure(ConsumerRecord<String, PaymentEvent> record, Exception error) {
        int currentAttempt = getRetryAttempt(record);
        int totalAttempts = 0;
        String nextTopic = null;

        for (RetryTier tier : TIERS) {
            if (currentAttempt < totalAttempts + tier.maxAttempts()) {
                nextTopic = record.topic().split("\\.retry")[0] + "." + tier.suffix();
                break;
            }
            totalAttempts += tier.maxAttempts();
        }

        if (nextTopic == null) {
            // Exhausted all tiers — send to DLT
            routeToDlt(record, error, currentAttempt);
            return;
        }

        ProducerRecord<String, PaymentEvent> retryRecord = new ProducerRecord<>(
            nextTopic, record.key(), record.value());
        retryRecord.headers()
            .add("x-retry-attempt", String.valueOf(currentAttempt + 1).getBytes())
            .add("x-original-topic", record.topic().getBytes())
            .add("x-error-message", error.getMessage().getBytes())
            .add("x-first-failure-time",
                 getOrDefault(record, "x-first-failure-time",
                     Instant.now().toString()).getBytes());

        kafkaTemplate.send(retryRecord);
    }

    private int getRetryAttempt(ConsumerRecord<String, ?> record) {
        var header = record.headers().lastHeader("x-retry-attempt");
        return header != null ? Integer.parseInt(new String(header.value())) : 0;
    }

    record RetryTier(String suffix, Duration delay, int maxAttempts) {}
}
```

---

## 🎯 Pattern 6: Exactly-Once Semantics (EOS)

### When Do You Actually Need EOS?

```
Ask yourself: What happens if a message is processed TWICE?

Scenario A: Deduct $100 from account → Duplicate = customer loses $200
  → NEED exactly-once (or idempotent consumer)

Scenario B: Send welcome email → Duplicate = customer gets 2 emails
  → TOLERABLE — at-least-once is fine

Scenario C: Update cache with latest price → Duplicate = same result
  → IDEMPOTENT — at-least-once is fine

MOST systems should use AT-LEAST-ONCE + IDEMPOTENT CONSUMERS
True EOS is expensive and adds complexity. Use it only for financial operations.
```

### Exactly-Once Producer + Consumer Config

```java
// ── Producer: Transactional ──
@Bean
public ProducerFactory<String, Object> producerFactory() {
    var props = new HashMap<String, Object>();
    props.put(ProducerConfig.ACKS_CONFIG, "all");
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "payment-tx-" + hostname());
    // Transactional ID must be unique per producer instance but stable across restarts

    return new DefaultKafkaProducerFactory<>(props);
}

@Bean
public KafkaTransactionManager<String, Object> kafkaTransactionManager(
        ProducerFactory<String, Object> pf) {
    return new KafkaTransactionManager<>(pf);
}

// ── Consumer: Read committed ──
@Bean
public ConsumerFactory<String, Object> consumerFactory() {
    var props = new HashMap<String, Object>();
    props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
    return new DefaultKafkaConsumerFactory<>(props);
}
```

### Idempotent Consumer Pattern (Preferred Over EOS)

```java
@Component
public class IdempotentPaymentConsumer {

    private final MongoTemplate mongoTemplate;

    @KafkaListener(topics = "payments.transaction.initiated")
    public void process(@Payload PaymentEvent event, Acknowledgment ack) {
        String eventId = event.eventId();

        // Atomic check-and-insert using MongoDB unique index
        try {
            mongoTemplate.insert(new ProcessedEvent(eventId, Instant.now()));
        } catch (DuplicateKeyException e) {
            log.info("Duplicate event detected, skipping: {}", eventId);
            ack.acknowledge();
            return; // Already processed — safe to skip
        }

        try {
            paymentService.process(event);
            ack.acknowledge();
        } catch (Exception e) {
            // Remove the processed marker so retry can re-attempt
            mongoTemplate.remove(Query.query(Criteria.where("_id").is(eventId)),
                ProcessedEvent.class);
            throw e;
        }
    }
}

// MongoDB collection with TTL index for auto-cleanup
@Document("processed_events")
@CompoundIndex(name = "ttl_idx", def = "{'processedAt': 1}", expireAfterSeconds = 86400)
public record ProcessedEvent(
    @Id String eventId,
    Instant processedAt
) {}
```

---

## 📦 Pattern 7: Message Design

### Event Types

```
1. EVENT NOTIFICATION (Thin)
   "Something happened" — consumer must query for details
   + Loose coupling, small messages
   - Extra network call to get details

2. EVENT-CARRIED STATE TRANSFER (Fat)
   "Something happened, here's everything you need"
   + Consumer is self-sufficient, no callback needed
   - Larger messages, tighter schema coupling

3. COMMAND
   "Please do this" — directed at a specific consumer
   + Clear intent, request-reply possible
   - Tighter coupling than events

Choose FAT events for: payment processing, cross-service data needs
Choose THIN events for: notifications, triggers, high-frequency events
```

### Event Envelope Pattern

```java
// Standard envelope — every event gets metadata
public record EventEnvelope<T>(
    String eventId,           // UUID — unique per event
    String eventType,         // "payment.initiated", "payment.completed"
    String aggregateId,       // Entity this event belongs to
    String aggregateType,     // "Payment", "Order"
    int version,              // Schema version for evolution
    Instant timestamp,        // When the event occurred
    String source,            // "payment-service"
    String correlationId,     // Trace across services
    String causationId,       // What caused this event
    Map<String, String> metadata,  // Extensible headers
    T payload                 // The actual event data
) {
    public EventEnvelope {
        Objects.requireNonNull(eventId, "eventId required");
        Objects.requireNonNull(eventType, "eventType required");
        Objects.requireNonNull(timestamp, "timestamp required");
    }
}

// Payload examples using sealed + records
public sealed interface PaymentEvent {
    String paymentId();

    record Initiated(String paymentId, Money amount, String customerId,
                     PaymentMethod method) implements PaymentEvent {}

    record Authorized(String paymentId, String authCode,
                      Instant authorizedAt) implements PaymentEvent {}

    record Captured(String paymentId, Money settledAmount,
                    String settlementId) implements PaymentEvent {}

    record Failed(String paymentId, String errorCode, String reason,
                  boolean retryable) implements PaymentEvent {}

    record Refunded(String paymentId, Money refundAmount,
                    String reason, String refundId) implements PaymentEvent {}
}
```

### Schema Evolution Rules

```
✅ SAFE changes (backward compatible):
  • Add optional field with default value
  • Add new event type to sealed interface
  • Deprecate field (keep in schema, stop populating)

⚠️ RISKY changes (need coordination):
  • Rename a field → add new, deprecate old, dual-write
  • Change field type → version the schema

❌ BREAKING changes (never in production):
  • Remove a required field
  • Change field type without versioning
  • Rename topic without migration

VERSIONING STRATEGY:
  v1: PaymentInitiated { amount: String }
  v2: PaymentInitiated { amount: Money }   ← new field type
  
  Approach: Topic-per-version OR envelope version field
    payments.transaction.initiated.v1  (deprecated, still consumed)
    payments.transaction.initiated.v2  (new consumers use this)
```

---

## 📊 Pattern 8: Consumer Group Strategies

### Consumer Group Patterns

```
Pattern A: COMPETING CONSUMERS (most common)
──────────────────────────────────────────────
Group: "payment-processor-v1"
  Consumer-1 → Partitions [0, 1, 2, 3]
  Consumer-2 → Partitions [4, 5, 6, 7]
  Consumer-3 → Partitions [8, 9, 10, 11]
Each message processed by ONE consumer. Scale by adding consumers.


Pattern B: FAN-OUT (multiple independent consumers)
──────────────────────────────────────────────
Topic: "payments.transaction.completed"
  Group "settlement-processor"  → ALL partitions (settles payment)
  Group "notification-sender"   → ALL partitions (sends receipt)
  Group "analytics-pipeline"    → ALL partitions (updates dashboard)
  Group "audit-logger"          → ALL partitions (compliance log)
Each message processed by ALL groups independently.


Pattern C: SELECTIVE CONSUMPTION
──────────────────────────────────────────────
Single topic with message type header:
  Header: x-event-type = "payment.initiated" | "payment.completed" | "payment.failed"

  Group "initiator-handler"  → filter: only "payment.initiated"
  Group "completion-handler" → filter: only "payment.completed"
  Group "failure-handler"    → filter: only "payment.failed"
Avoids topic proliferation, but all consumers poll all messages.
```

### Rebalancing Strategy

```yaml
# application.yml
spring:
  kafka:
    consumer:
      properties:
        # Cooperative rebalancing — no stop-the-world
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor

        # Static membership — reduces rebalancing during rolling deployments
        group.instance.id: ${HOSTNAME}   # Stable ID across restarts
        session.timeout.ms: 60000        # Higher timeout for static members
```

### Consumer Lag Monitoring

```java
@Component
public class ConsumerLagMonitor {

    private final MeterRegistry meterRegistry;

    @KafkaListener(topics = "payments.transaction.initiated")
    public void process(@Payload PaymentEvent event,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset,
                        @Header(KafkaHeaders.TIMESTAMP_TYPE) String tsType,
                        @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long eventTimestamp,
                        Acknowledgment ack) {

        // Track processing lag (time between event creation and processing)
        long lagMs = System.currentTimeMillis() - eventTimestamp;
        meterRegistry.timer("kafka.consumer.processing.lag",
            "topic", "payments.transaction.initiated",
            "partition", String.valueOf(partition))
            .record(Duration.ofMillis(lagMs));

        if (lagMs > 60000) {
            log.warn("High consumer lag: {}ms on partition {}", lagMs, partition);
        }

        process(event);
        ack.acknowledge();
    }
}
```

---

## 🔄 Pattern 9: Kafka Streams Patterns

### Stateless Transformations

```java
StreamsBuilder builder = new StreamsBuilder();

// Filter + Map + Route
builder.<String, PaymentEvent>stream("payments.transaction.all")
    // Filter high-value payments
    .filter((key, event) -> event.amount().compareTo(new BigDecimal("10000")) >= 0)
    // Enrich with risk score
    .mapValues(event -> new EnrichedPayment(event, riskEngine.score(event)))
    // Route to different topics based on risk
    .split()
    .branch((key, enriched) -> enriched.riskScore() > 80,
        Branched.withConsumer(s -> s.to("payments.high-risk")))
    .branch((key, enriched) -> enriched.riskScore() > 40,
        Branched.withConsumer(s -> s.to("payments.medium-risk")))
    .defaultBranch(
        Branched.withConsumer(s -> s.to("payments.low-risk")));
```

### Stateful: Aggregation & Windowed Processing

```java
// Real-time fraud detection: flag if > 5 transactions in 10 minutes per customer
builder.<String, PaymentEvent>stream("payments.transaction.initiated")
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(10)))
    .count(Materialized.as("transaction-count-store"))
    .filter((windowedKey, count) -> count > 5)
    .toStream()
    .mapValues((windowedKey, count) ->
        new FraudAlert(windowedKey.key(), count, windowedKey.window().startTime()))
    .to("fraud.alerts.velocity");
```

### Join Patterns

```java
// Stream-Stream join: Match payment with fraud check within 30s window
KStream<String, PaymentEvent> payments =
    builder.stream("payments.transaction.initiated");
KStream<String, FraudResult> fraudChecks =
    builder.stream("fraud.check.completed");

payments.join(
    fraudChecks,
    (payment, fraud) -> new PaymentDecision(payment, fraud),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofSeconds(30)),
    StreamJoined.with(Serdes.String(), paymentSerde, fraudSerde)
)
.to("payments.transaction.decided");


// Stream-Table join: Enrich payment with current account data
KTable<String, Account> accounts =
    builder.table("accounts.balance.snapshot");  // Compacted topic

payments
    .selectKey((key, payment) -> payment.accountId())  // Re-key to match account key
    .join(accounts,
        (payment, account) -> new EnrichedPayment(payment, account))
    .to("payments.transaction.enriched");
```

---

## 🌍 Pattern 10: Multi-Region Kafka

### Topologies

```
ACTIVE-PASSIVE
───────────────
Region A (Active):  Produces + Consumes
Region B (Passive): MirrorMaker 2 replicates from A
                    Consumers dormant — activated on failover
Use: DR only, simpler, lower cost

ACTIVE-ACTIVE
───────────────
Region A: Produces + Consumes LOCAL topics
Region B: Produces + Consumes LOCAL topics
MirrorMaker 2: Bidirectional replication with topic prefixes
  A → B: payments.transaction.initiated → regionA.payments.transaction.initiated
  B → A: payments.transaction.initiated → regionB.payments.transaction.initiated
Use: Low latency globally, complex conflict resolution

AGGREGATE
───────────────
Region A, B, C: Each produces to local Kafka
Central region: MirrorMaker 2 aggregates all regions
  → regionA.payments.*, regionB.payments.*, regionC.payments.*
  → Central analytics, reporting, compliance
Use: Centralized analytics, audit
```

### MirrorMaker 2 Configuration

```properties
# mm2.properties
clusters = primary, secondary
primary.bootstrap.servers = kafka-primary-1:9092,kafka-primary-2:9092
secondary.bootstrap.servers = kafka-secondary-1:9092,kafka-secondary-2:9092

# Replication flow
primary->secondary.enabled = true
primary->secondary.topics = payments\..*,accounts\..*,fraud\..*
primary->secondary.topics.exclude = .*\.dlt,.*\.retry.*,.*internal.*

# Consumer group offset sync
primary->secondary.groups = payment-processor.*,settlement.*
primary->secondary.emit.checkpoints.enabled = true
primary->secondary.sync.group.offsets.enabled = true

# Heartbeats for monitoring
primary->secondary.emit.heartbeats.enabled = true
heartbeats.topic.replication.factor = 3

# Replication settings
replication.factor = 3
offset.flush.interval.ms = 10000
```

---

## ⚙️ Pattern 11: Production Configuration Templates

### application.yml — Complete Production Config

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}

    properties:
      # Security (production)
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${KAFKA_USER}" password="${KAFKA_PASSWORD}";
      ssl.truststore.location: ${KAFKA_TRUSTSTORE_PATH}
      ssl.truststore.password: ${KAFKA_TRUSTSTORE_PASSWORD}

    producer:
      acks: all
      retries: 2147483647
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        delivery.timeout.ms: 120000
        linger.ms: 20
        batch.size: 32768
        compression.type: lz4
        buffer.memory: 67108864
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

    consumer:
      group-id: ${spring.application.name}-${DEPLOYMENT_ENV:local}
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        max.poll.records: 100
        max.poll.interval.ms: 300000
        session.timeout.ms: 30000
        heartbeat.interval.ms: 10000
        isolation.level: read_committed
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

    listener:
      ack-mode: manual
      concurrency: ${KAFKA_CONSUMER_CONCURRENCY:6}
      type: single
```

### Health Check & Metrics

```java
@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaTemplate<String, ?> kafkaTemplate;

    @Override
    public Health health() {
        try (var admin = AdminClient.create(
                kafkaTemplate.getProducerFactory().getConfigurationProperties())) {

            var clusterInfo = admin.describeCluster();
            var nodes = clusterInfo.nodes().get(5, TimeUnit.SECONDS);
            var controllerId = clusterInfo.controller().get(5, TimeUnit.SECONDS);

            return Health.up()
                .withDetail("clusterId", clusterInfo.clusterId().get())
                .withDetail("nodeCount", nodes.size())
                .withDetail("controller", controllerId.idString())
                .build();

        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 🚫 Kafka Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Auto-commit enabled** | Messages lost on crash before processing | `enable.auto.commit=false` + manual ack |
| **No dead-letter topic** | Poison pills block entire partition | Always configure DLT |
| **Huge messages (>1MB)** | Broker memory pressure, slow replication | Claim-check pattern, compress, or chunk |
| **Using Kafka as a database** | Not designed for random access | Use compacted topics + materialized views |
| **Single partition for ordering** | Zero parallelism, bottleneck | Key-based partitioning for per-entity ordering |
| **Consumer group per instance** | Every instance gets ALL messages | Shared group ID for competing consumers |
| **Ignoring consumer lag** | Silent data loss, stale processing | Monitor lag, alert on threshold |
| **Processing without idempotency** | Duplicates on rebalance or retry | Idempotency key + dedup check |
| **Synchronous produce in request path** | Latency spike blocks HTTP response | Async produce or outbox pattern |
| **No schema evolution strategy** | Breaking changes crash consumers | Envelope versioning or schema registry |
| **Retry topic without delay** | Tight retry loop hammers downstream | Exponential backoff across retry tiers |
| **Committing before processing** | Message lost if processing fails | Process THEN commit |
| **`latest` offset reset** | Misses messages produced during downtime | `earliest` + idempotent consumer |
| **No monitoring/alerting** | Problems discovered by customers | Lag, throughput, error rate dashboards |

---

## 📏 Quick Reference: Configuration Cheat Table

| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| **Replication factor** | 1 | 3 | 3 |
| **min.insync.replicas** | 1 | 2 | 2 |
| **acks** | 1 | all | all |
| **enable.idempotence** | false | true | true |
| **compression** | none | lz4 | lz4 |
| **Consumer concurrency** | 1 | 3 | partition count |
| **max.poll.records** | 10 | 50 | 100-500 |
| **session.timeout.ms** | 10000 | 30000 | 30000 |
| **SSL/SASL** | off | on | on |
| **Monitoring** | console logs | basic metrics | full observability |

---

## 💡 Golden Rules of Kafka

```
1.  EVERY external call gets a timeout; every Kafka consumer gets a DLT.
2.  Ordering is per-partition — design your keys to match your ordering needs.
3.  Idempotent consumers beat exactly-once semantics in 95% of cases.
4.  Manual offset commit AFTER successful processing — never before.
5.  Consumer group ID is your scaling unit — same group = competing, different group = fan-out.
6.  Partition count can go UP but never DOWN — choose wisely at creation.
7.  Message size should be < 100KB typically, < 1MB max — use claim-check for large payloads.
8.  Schema evolution is a contract — treat topics like APIs with versioning.
9.  Monitor consumer lag like you monitor error rates — it's your early warning system.
10. Test with Testcontainers + embedded Kafka — never mock the broker in integration tests.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Spring Kafka 3.x / Confluent Platform*
