# Apache Kafka Deep Dive

A comprehensive guide to Kafka internals, advanced patterns, performance tuning, and production operations for enterprise event-driven systems.

---

## Table of Contents

1. [Kafka Architecture](#kafka-architecture)
2. [Topics & Partitions](#topics--partitions)
3. [Producer Internals](#producer-internals)
4. [Consumer Internals](#consumer-internals)
5. [Consumer Groups](#consumer-groups)
6. [Exactly-Once Semantics](#exactly-once-semantics)
7. [Kafka Streams](#kafka-streams)
8. [Performance Tuning](#performance-tuning)
9. [Monitoring & Operations](#monitoring--operations)
10. [Disaster Recovery](#disaster-recovery)
11. [Security](#security)
12. [Production Patterns](#production-patterns)

---

## Kafka Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────┐
│                    Kafka Cluster                         │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │             │
│  │          │  │          │  │          │             │
│  │ Topic A  │  │ Topic A  │  │ Topic A  │             │
│  │ Part 0   │  │ Part 1   │  │ Part 2   │             │
│  │ (Leader) │  │(Follower)│  │(Follower)│             │
│  │          │  │          │  │          │             │
│  │ Topic B  │  │ Topic B  │  │ Topic B  │             │
│  │ Part 1   │  │ Part 0   │  │ Part 2   │             │
│  │(Follower)│  │ (Leader) │  │(Follower)│             │
│  └──────────┘  └──────────┘  └──────────┘             │
│                                                          │
│  ┌──────────────────────────────────────────┐          │
│  │         ZooKeeper / KRaft                 │          │
│  │  - Cluster metadata                       │          │
│  │  - Leader election                        │          │
│  │  - Configuration                          │          │
│  └──────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘

Producers ──write──> Brokers ──read──> Consumers
```

### Log Structure

```
Partition (Ordered, Immutable Log):

Offset: 0      1      2      3      4      5      6
      ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
      │ Msg1 │ Msg2 │ Msg3 │ Msg4 │ Msg5 │ Msg6 │ Msg7 │
      └──────┴──────┴──────┴──────┴──────┴──────┴──────┘
        ▲                                           ▲
        │                                           │
    Committed                              Current Offset
    Offset (3)                             (High Water Mark: 7)

Properties:
- Append-only (writes at the end)
- Immutable (messages never change)
- Ordered within partition
- Distributed across brokers
```

---

## Topics & Partitions

### Partitioning Strategy

```java
// Default partitioner - hash of key
@Service
public class PaymentProducer {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    public void publishPayment(PaymentEvent event) {
        // Key-based partitioning - same key → same partition
        kafkaTemplate.send(
            "payment-events",
            event.getUserId(), // Key - ensures order per user
            event
        );
    }
}

// Custom partitioner
public class PaymentPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes,
                        Cluster cluster) {

        PaymentEvent event = (PaymentEvent) value;

        // VIP users get dedicated partition for priority
        if (event.isVip()) {
            return 0;
        }

        // Regular users distributed across remaining partitions
        int numPartitions = cluster.partitionCountForTopic(topic);
        return (Math.abs(key.hashCode()) % (numPartitions - 1)) + 1;
    }
}
```

### Partition Count

```yaml
# Choosing partition count
# Factor 1: Throughput target
partitions_needed = target_throughput / consumer_throughput

# Factor 2: Parallelism
partitions = max_consumers_in_group

# Factor 3: Broker capacity
partitions_per_broker = total_partitions / broker_count

# Example:
# Target: 100,000 msg/s
# Consumer: 10,000 msg/s
# Result: 10 partitions minimum

# Production recommendation:
# - Start with: max(target_consumers, throughput_based)
# - Cannot decrease (only increase)
# - More partitions = more overhead
```

---

## Producer Internals

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, PaymentEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        // Basic config
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "kafka1:9092,kafka2:9092,kafka3:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            JsonSerializer.class);

        // Reliability
        config.put(ProducerConfig.ACKS_CONFIG, "all"); // Wait for all replicas
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        // Performance
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        config.put(ProducerConfig.LINGER_MS_CONFIG, 100); // Batch delay
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768); // 32KB

        // Timeout
        config.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
        config.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### Producer Batching

```java
@Service
public class OptimizedPaymentProducer {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    // Async send with callback
    public CompletableFuture<SendResult<String, PaymentEvent>> sendAsync(
            PaymentEvent event) {

        return kafkaTemplate.send("payment-events", event.getUserId(), event)
            .thenApply(result -> {
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("Sent to partition {} at offset {}",
                    metadata.partition(), metadata.offset());
                return result;
            })
            .exceptionally(ex -> {
                log.error("Failed to send payment event", ex);
                throw new PaymentEventPublishException(ex);
            });
    }

    // Batch send
    public List<CompletableFuture<SendResult<String, PaymentEvent>>>
            sendBatch(List<PaymentEvent> events) {

        return events.stream()
            .map(this::sendAsync)
            .toList();
    }
}
```

---

## Consumer Internals

### Consumer Configuration

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, PaymentEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "kafka1:9092,kafka2:9092");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            JsonDeserializer.class);

        // Consumer group
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "payment-processor");

        // Offset management
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // Manual

        // Performance
        config.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024); // 1KB min
        config.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500); // Wait 500ms
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);

        // Session
        config.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        config.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5min

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

### Manual Offset Management

```java
@Service
public class PaymentEventConsumer {

    @KafkaListener(topics = "payment-events",
                   containerFactory = "kafkaListenerContainerFactory")
    public void consume(
            @Payload PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        try {
            // Process event
            processPayment(event);

            // Commit offset manually
            ack.acknowledge();

            log.info("Processed payment from partition {} at offset {}",
                partition, offset);

        } catch (Exception e) {
            log.error("Failed to process payment event", e);
            // Don't acknowledge - will retry
            throw e;
        }
    }

    @Transactional
    private void processPayment(PaymentEvent event) {
        // Business logic
        Payment payment = paymentService.process(event);

        // Store processed event ID for idempotency
        processedEventRepository.save(
            new ProcessedEvent(event.getId(), Instant.now())
        );
    }
}
```

---

## Consumer Groups

### Rebalancing

```
Initial State (3 consumers, 6 partitions):

Consumer 1: [P0, P1]
Consumer 2: [P2, P3]
Consumer 3: [P4, P5]

After Consumer 2 Crashes (Rebalance):

Consumer 1: [P0, P1, P2]
Consumer 3: [P3, P4, P5]

After Consumer 4 Joins (Rebalance):

Consumer 1: [P0, P1]
Consumer 3: [P2, P3]
Consumer 4: [P4, P5]

Rebalance Types:
1. Eager Rebalance (Stop-The-World)
   - Stop all consumers
   - Reassign partitions
   - Resume all consumers

2. Cooperative Rebalance (Incremental)
   - Only affected partitions reassigned
   - Other consumers continue processing
```

### Rebalance Listener

```java
@Service
public class RebalanceAwareConsumer {

    @KafkaListener(topics = "payment-events")
    public void listen(
            ConsumerRecord<String, PaymentEvent> record,
            Acknowledgment ack,
            Consumer<?, ?> consumer) {

        // Process message
        processPayment(record.value());
        ack.acknowledge();
    }

    @Bean
    public ConsumerRebalanceListener rebalanceListener() {
        return new ConsumerRebalanceListener() {

            @Override
            public void onPartitionsRevoked(
                    Collection<TopicPartition> partitions) {
                log.warn("Partitions revoked: {}", partitions);
                // Clean up resources, commit offsets, etc.
                cleanup();
            }

            @Override
            public void onPartitionsAssigned(
                    Collection<TopicPartition> partitions) {
                log.info("Partitions assigned: {}", partitions);
                // Initialize resources
                initialize();
            }
        };
    }
}
```

---

## Exactly-Once Semantics

### Transactional Producer

```java
@Configuration
public class TransactionalKafkaConfig {

    @Bean
    public ProducerFactory<String, PaymentEvent> transactionalProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        // ... other config

        // Enable transactions
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "payment-tx-");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        return new DefaultKafkaProducerFactory<>(config);
    }
}

@Service
public class TransactionalPaymentService {

    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    private final PaymentRepository paymentRepository;

    @Transactional
    public void processPayment(PaymentRequest request) {
        // Database transaction
        Payment payment = paymentRepository.save(
            Payment.from(request)
        );

        // Kafka transaction (atomic with DB)
        kafkaTemplate.executeInTransaction(ops -> {
            ops.send("payment-events",
                new PaymentCreatedEvent(payment.getId()));
            ops.send("notification-events",
                new SendNotificationEvent(payment.getUserId()));
            return true;
        });

        // Both DB and Kafka commit together
        // Or both rollback together
    }
}
```

### Read-Process-Write Pattern

```java
@Service
public class ExactlyOnceConsumer {

    @KafkaListener(topics = "input-topic")
    @Transactional("kafkaTransactionManager")
    public void consumeTransform(
            @Payload PaymentEvent inputEvent,
            Acknowledgment ack) {

        // Read from input topic
        // Process
        EnrichedPaymentEvent enrichedEvent = enrich(inputEvent);

        // Write to output topic (transactionally)
        kafkaTemplate.send("output-topic", enrichedEvent);

        // Commit offset (transactionally)
        ack.acknowledge();

        // All or nothing:
        // - Offset committed
        // - Output message sent
        // - Or both fail
    }
}
```

---

## Kafka Streams

### Stream Processing

```java
@Configuration
@EnableKafkaStreams
public class KafkaStreamsConfig {

    @Bean
    public StreamsBuilder streamsBuilder() {
        return new StreamsBuilder();
    }

    @Bean
    public KStream<String, PaymentEvent> paymentStream(
            StreamsBuilder builder) {

        // Source stream
        KStream<String, PaymentEvent> payments = builder
            .stream("payment-events",
                Consumed.with(Serdes.String(), paymentEventSerde()));

        // Filter
        KStream<String, PaymentEvent> highValuePayments = payments
            .filter((key, payment) ->
                payment.getAmount().compareTo(new BigDecimal("1000")) > 0
            );

        // Transform
        KStream<String, PaymentProcessedEvent> processed = payments
            .mapValues(payment -> processPayment(payment));

        // Aggregate
        KTable<String, Long> paymentCountByUser = payments
            .groupBy((key, payment) -> payment.getUserId())
            .count(Materialized.as("payment-counts"));

        // Join
        KStream<String, EnrichedPayment> enriched = payments
            .join(
                userTable,
                (payment, user) -> new EnrichedPayment(payment, user),
                Joined.with(Serdes.String(), paymentSerde(), userSerde())
            );

        // Sink
        processed.to("payment-processed",
            Produced.with(Serdes.String(), processedEventSerde()));

        return payments;
    }
}
```

---

## Performance Tuning

### Producer Performance

```java
// High throughput configuration
@Bean
public ProducerFactory<String, PaymentEvent> highThroughputProducer() {
    Map<String, Object> config = new HashMap<>();

    // Batching
    config.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536); // 64KB
    config.put(ProducerConfig.LINGER_MS_CONFIG, 100); // Wait 100ms

    // Compression
    config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

    // Buffer
    config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864); // 64MB

    // Parallelism
    config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

    return new DefaultKafkaProducerFactory<>(config);
}
```

### Consumer Performance

```java
// High throughput consumer
@Bean
public ConsumerFactory<String, PaymentEvent> highThroughputConsumer() {
    Map<String, Object> config = new HashMap<>();

    // Fetch more data
    config.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 10240); // 10KB
    config.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
    config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 1000);

    // Parallel processing
    config.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG,
        1048576); // 1MB per partition

    return new DefaultKafkaConsumerFactory<>(config);
}

// Concurrent consumer
@Bean
public ConcurrentKafkaListenerContainerFactory<String, PaymentEvent>
        kafkaListenerContainerFactory() {

    ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> factory =
        new ConcurrentKafkaListenerContainerFactory<>();

    factory.setConsumerFactory(consumerFactory());
    factory.setConcurrency(3); // 3 consumer threads
    factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);

    return factory;
}
```

---

## Monitoring & Operations

### Key Metrics

```yaml
# Producer Metrics
- record-send-rate: Messages sent per second
- record-error-rate: Failed sends per second
- request-latency-avg: Average request time
- buffer-available-bytes: Buffer space available

# Consumer Metrics
- records-consumed-rate: Messages consumed per second
- fetch-latency-avg: Fetch request latency
- records-lag: Offset lag
- commit-latency-avg: Offset commit time

# Broker Metrics
- bytes-in-per-sec: Incoming data rate
- bytes-out-per-sec: Outgoing data rate
- under-replicated-partitions: Partitions not fully replicated
- offline-partitions: Partitions without leader
```

### Monitoring with Micrometer

```java
@Component
public class KafkaMetrics {

    private final MeterRegistry registry;

    @Autowired
    public KafkaMetrics(MeterRegistry registry) {
        this.registry = registry;
    }

    public void recordMessageProcessed(String topic, long duration) {
        Timer.builder("kafka.message.processed")
            .tag("topic", topic)
            .register(registry)
            .record(Duration.ofMillis(duration));
    }

    public void recordConsumerLag(String topic, int partition, long lag) {
        Gauge.builder("kafka.consumer.lag", () -> lag)
            .tag("topic", topic)
            .tag("partition", String.valueOf(partition))
            .register(registry);
    }
}
```

---

## Production Best Practices

### ✅ DO

- Use idempotent producers
- Enable compression (lz4 or snappy)
- Monitor consumer lag
- Set appropriate partition count
- Use consumer groups for scaling
- Implement proper error handling
- Use schema registry
- Configure replication factor ≥ 3
- Set up monitoring and alerting
- Plan for disaster recovery

### ❌ DON'T

- Use auto-commit with at-least-once
- Create too many partitions (overhead)
- Ignore rebalance events
- Skip offset management strategy
- Use single-broker clusters in production
- Delete topics without backup
- Ignore under-replicated partitions
- Process messages synchronously in tight loop
- Use default configurations in production

---

*This guide provides comprehensive patterns for operating Apache Kafka at scale. Kafka is powerful but requires careful configuration and monitoring for production success.*
