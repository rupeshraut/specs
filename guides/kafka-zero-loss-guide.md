# Effective Kafka Zero Message Loss

A comprehensive guide to implementing zero message loss in Kafka with error handling, retry strategies, and recovery patterns for Spring Kafka and vanilla Kafka.

---

## Table of Contents

1. [Understanding Message Loss](#understanding-message-loss)
2. [Producer Configuration for Zero Loss](#producer-configuration-for-zero-loss)
3. [Producer Idempotence](#producer-idempotence)
4. [Transactions](#transactions)
5. [Consumer Configuration](#consumer-configuration)
6. [Manual Offset Management](#manual-offset-management)
7. [Error Handling Strategies](#error-handling-strategies)
8. [Retry Patterns](#retry-patterns)
9. [Dead Letter Queue (DLQ)](#dead-letter-queue-dlq)
10. [Exactly-Once Semantics](#exactly-once-semantics)
11. [Recovery Patterns](#recovery-patterns)
12. [Spring Kafka Implementation](#spring-kafka-implementation)
13. [Vanilla Kafka Implementation](#vanilla-kafka-implementation)
14. [Monitoring and Alerting](#monitoring-and-alerting)

---

## Understanding Message Loss

### Where Messages Can Be Lost

```java
/**
 * Message Loss Scenarios:
 * 
 * 1. PRODUCER SIDE:
 *    - Network failures before ack
 *    - Producer crash before send
 *    - Broker not responding
 *    - Buffer overflow
 * 
 * 2. BROKER SIDE:
 *    - Unclean leader election
 *    - Broker crash before replication
 *    - Disk failure
 * 
 * 3. CONSUMER SIDE:
 *    - Commit offset before processing
 *    - Consumer crash during processing
 *    - Processing error without retry
 *    - Rebalance during processing
 * 
 * Prevention Strategy:
 * ┌─────────────────────────────────────────┐
 * │ Producer: acks=all, retries, idempotence│
 * │ Broker: replication.factor >= 3         │
 * │ Consumer: manual commit, retry, DLQ     │
 * └─────────────────────────────────────────┘
 */
```

---

## Producer Configuration for Zero Loss

### Critical Producer Settings

```yaml
# Spring Boot application.yml
spring:
  kafka:
    producer:
      # CRITICAL: Wait for all replicas to acknowledge
      acks: all  # or "-1"
      
      # Enable idempotence (prevents duplicates)
      properties:
        enable.idempotence: true
      
      # Retries
      retries: 2147483647  # Integer.MAX_VALUE
      
      # Max in-flight requests per connection
      # Must be <= 5 for idempotence
      max-in-flight-requests-per-connection: 5
      
      # Compression (optional, reduces network load)
      compression-type: lz4
      
      # Buffer settings
      buffer-memory: 33554432  # 32 MB
      
      # Batch settings
      batch-size: 16384
      linger-ms: 10
      
      # Request timeout
      properties:
        request.timeout.ms: 30000
        delivery.timeout.ms: 120000
```

### Java Configuration (Non-Spring)

```java
public class ZeroLossProducerConfig {
    
    public static Properties getProducerConfig() {
        Properties props = new Properties();
        
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        // CRITICAL SETTINGS FOR ZERO LOSS
        props.put(ProducerConfig.ACKS_CONFIG, "all");  // Wait for all ISR
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // No duplicates
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);  // Unlimited retries
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        // Timeouts
        props.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
        props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);
        
        // Serializers
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        
        return props;
    }
}
```

### Broker Configuration

```properties
# server.properties

# Replication factor >= 3 for production
default.replication.factor=3

# Min in-sync replicas >= 2
min.insync.replicas=2

# Disable unclean leader election (prevents data loss)
unclean.leader.election.enable=false

# Replica lag time
replica.lag.time.max.ms=10000

# Log flush settings (optional, impacts performance)
log.flush.interval.messages=10000
log.flush.interval.ms=1000
```

---

## Producer Idempotence

### Enable Idempotence

```java
/**
 * Idempotent Producer prevents duplicate messages
 * 
 * Guarantees:
 * - Exactly-once delivery to a partition
 * - Automatic deduplication
 * - No message reordering
 * 
 * Requirements:
 * - enable.idempotence=true
 * - acks=all
 * - retries > 0
 * - max.in.flight.requests.per.connection <= 5
 */

@Configuration
public class IdempotentProducerConfig {
    
    @Bean
    public ProducerFactory<String, Event> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}

// Vanilla Kafka
public class IdempotentProducer {
    
    private final KafkaProducer<String, String> producer;
    
    public IdempotentProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        
        this.producer = new KafkaProducer<>(props);
    }
    
    public void send(String topic, String key, String value) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
        try {
            RecordMetadata metadata = producer.send(record).get();
            System.out.println("Sent to partition: " + metadata.partition());
        } catch (Exception e) {
            // Handle error - message will be retried automatically
            System.err.println("Send failed: " + e.getMessage());
        }
    }
}
```

---

## Transactions

### Transactional Producer

```java
/**
 * Transactional Producer provides:
 * - Atomic writes across multiple partitions
 * - Exactly-once semantics
 * - Read-committed isolation
 */

// Spring Kafka
@Configuration
public class TransactionalProducerConfig {
    
    @Bean
    public ProducerFactory<String, Event> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-producer-1");
        
        DefaultKafkaProducerFactory<String, Event> factory = 
            new DefaultKafkaProducerFactory<>(config);
        
        return factory;
    }
    
    @Bean
    public KafkaTemplate<String, Event> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

@Service
public class TransactionalService {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    
    @Transactional  // Spring transaction management
    public void processWithTransaction(List<Event> events) {
        kafkaTemplate.executeInTransaction(operations -> {
            events.forEach(event -> {
                operations.send("topic1", event);
                operations.send("topic2", event);
            });
            return true;
        });
    }
}

// Vanilla Kafka
public class VanillaTransactionalProducer {
    
    private final KafkaProducer<String, String> producer;
    
    public VanillaTransactionalProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-producer-1");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        this.producer = new KafkaProducer<>(props);
        producer.initTransactions();  // Initialize once
    }
    
    public void sendTransactional(List<ProducerRecord<String, String>> records) {
        try {
            producer.beginTransaction();
            
            for (ProducerRecord<String, String> record : records) {
                producer.send(record);
            }
            
            producer.commitTransaction();
            
        } catch (Exception e) {
            producer.abortTransaction();
            throw new RuntimeException("Transaction failed", e);
        }
    }
}
```

---

## Consumer Configuration

### Zero-Loss Consumer Settings

```yaml
# Spring Boot application.yml
spring:
  kafka:
    consumer:
      # CRITICAL: Manual offset management
      enable-auto-commit: false
      
      # Isolation level for transactional messages
      properties:
        isolation.level: read_committed
      
      # Session and heartbeat
      session-timeout-ms: 10000
      heartbeat-interval-ms: 3000
      
      # Max poll
      max-poll-records: 100
      max-poll-interval-ms: 300000
      
      # Auto offset reset (for new consumer groups)
      auto-offset-reset: earliest
```

### Java Configuration

```java
public class ZeroLossConsumerConfig {
    
    public static Properties getConsumerConfig() {
        Properties props = new Properties();
        
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "zero-loss-group");
        
        // CRITICAL SETTINGS
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // Manual commit
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");  // Transactions
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        // Deserializers
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        
        return props;
    }
}
```

---

## Manual Offset Management

### Spring Kafka - Manual Commit

```java
@Configuration
public class ManualCommitConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            manualCommitFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        return factory;
    }
}

@Service
public class ZeroLossConsumer {
    
    @KafkaListener(
        topics = "events",
        containerFactory = "manualCommitFactory"
    )
    public void consume(
            ConsumerRecord<String, Event> record,
            Acknowledgment ack) {
        
        try {
            // Process message
            processEvent(record.value());
            
            // Persist to database
            eventRepository.save(record.value());
            
            // COMMIT ONLY AFTER SUCCESSFUL PROCESSING
            ack.acknowledge();
            
        } catch (Exception e) {
            // Don't commit - message will be reprocessed
            logger.error("Processing failed, will retry", e);
            // Optionally: send to DLQ after max retries
        }
    }
}
```

### Vanilla Kafka - Manual Commit

```java
public class VanillaManualCommitConsumer {
    
    private final KafkaConsumer<String, String> consumer;
    
    public VanillaManualCommitConsumer() {
        Properties props = ZeroLossConsumerConfig.getConsumerConfig();
        this.consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("events"));
    }
    
    public void consume() {
        try {
            while (true) {
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    try {
                        // Process message
                        processMessage(record.value());
                        
                        // Commit this specific offset
                        Map<TopicPartition, OffsetAndMetadata> offsets = 
                            Collections.singletonMap(
                                new TopicPartition(record.topic(), record.partition()),
                                new OffsetAndMetadata(record.offset() + 1)
                            );
                        
                        consumer.commitSync(offsets);
                        
                    } catch (Exception e) {
                        logger.error("Processing failed", e);
                        // Don't commit - message will be reprocessed
                    }
                }
            }
        } finally {
            consumer.close();
        }
    }
    
    private void processMessage(String message) {
        // Business logic
    }
}
```

---

## Error Handling Strategies

### Try-Catch with Logging

```java
@Service
public class ErrorHandlingConsumer {
    
    @KafkaListener(topics = "events", containerFactory = "manualCommitFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            processEvent(record.value());
            ack.acknowledge();
            
        } catch (TransientException e) {
            // Transient error - don't commit, will retry
            logger.warn("Transient error, will retry: {}", e.getMessage());
            
        } catch (PermanentException e) {
            // Permanent error - send to DLQ and commit
            logger.error("Permanent error, sending to DLQ", e);
            sendToDLQ(record, e);
            ack.acknowledge();
            
        } catch (Exception e) {
            // Unknown error - be conservative, don't commit
            logger.error("Unknown error, will retry", e);
        }
    }
    
    private void sendToDLQ(ConsumerRecord<String, Event> record, Exception e) {
        FailedEvent failedEvent = new FailedEvent(
            record.value(),
            e.getMessage(),
            Instant.now()
        );
        kafkaTemplate.send("events.dlq", failedEvent);
    }
}
```

---

## Retry Patterns

### Immediate Retry with Counter

```java
@Service
public class ImmediateRetryConsumer {
    
    private static final int MAX_RETRIES = 3;
    
    @KafkaListener(topics = "events", containerFactory = "manualCommitFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        Event event = record.value();
        int retryCount = getRetryCount(event);
        
        try {
            processEvent(event);
            ack.acknowledge();
            
        } catch (Exception e) {
            if (retryCount < MAX_RETRIES) {
                logger.warn("Retry {} of {}", retryCount + 1, MAX_RETRIES);
                
                // Immediate retry
                try {
                    Thread.sleep(1000 * (retryCount + 1));  // Backoff
                    event.setRetryCount(retryCount + 1);
                    processEvent(event);
                    ack.acknowledge();
                } catch (Exception retryError) {
                    logger.error("Retry failed", retryError);
                    // Don't commit - will be reprocessed on next poll
                }
            } else {
                logger.error("Max retries exceeded, sending to DLQ");
                sendToDLQ(record, e);
                ack.acknowledge();
            }
        }
    }
}
```

### Retry Topic Pattern

```java
@Service
public class RetryTopicConsumer {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    
    // Main topic consumer
    @KafkaListener(topics = "events", containerFactory = "manualCommitFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            processEvent(record.value());
            ack.acknowledge();
            
        } catch (Exception e) {
            Event event = record.value();
            int retryCount = getRetryCount(event);
            
            if (retryCount < 3) {
                // Send to retry topic
                event.setRetryCount(retryCount + 1);
                kafkaTemplate.send("events.retry", event);
                ack.acknowledge();
                logger.warn("Sent to retry topic, attempt {}", retryCount + 1);
            } else {
                // Max retries - send to DLQ
                sendToDLQ(record, e);
                ack.acknowledge();
            }
        }
    }
    
    // Retry topic consumer (with delay)
    @KafkaListener(topics = "events.retry", containerFactory = "manualCommitFactory")
    public void consumeRetry(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            // Add delay before retry
            Thread.sleep(5000);
            
            processEvent(record.value());
            ack.acknowledge();
            
        } catch (Exception e) {
            Event event = record.value();
            int retryCount = getRetryCount(event);
            
            if (retryCount < 3) {
                event.setRetryCount(retryCount + 1);
                kafkaTemplate.send("events.retry", event);
                ack.acknowledge();
            } else {
                sendToDLQ(record, e);
                ack.acknowledge();
            }
        }
    }
}
```

### Exponential Backoff Retry

```java
@Service
public class ExponentialBackoffRetry {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    private final Map<Integer, String> retryTopics = Map.of(
        1, "events.retry.1s",
        2, "events.retry.2s",
        3, "events.retry.4s",
        4, "events.retry.8s"
    );
    
    @KafkaListener(topics = "events", containerFactory = "manualCommitFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            processEvent(record.value());
            ack.acknowledge();
            
        } catch (Exception e) {
            Event event = record.value();
            int retryCount = getRetryCount(event);
            
            if (retryCount < 4) {
                String retryTopic = retryTopics.get(retryCount + 1);
                event.setRetryCount(retryCount + 1);
                kafkaTemplate.send(retryTopic, event);
                ack.acknowledge();
                
                logger.warn("Sent to {} (backoff: {}s)", 
                    retryTopic, Math.pow(2, retryCount));
            } else {
                sendToDLQ(record, e);
                ack.acknowledge();
            }
        }
    }
    
    // Consumers for each retry level
    @KafkaListener(topics = "events.retry.1s")
    public void retry1s(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        retryWithDelay(record, ack, 1000);
    }
    
    @KafkaListener(topics = "events.retry.2s")
    public void retry2s(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        retryWithDelay(record, ack, 2000);
    }
    
    private void retryWithDelay(
            ConsumerRecord<String, Event> record,
            Acknowledgment ack,
            long delayMs) {
        
        try {
            Thread.sleep(delayMs);
            processEvent(record.value());
            ack.acknowledge();
        } catch (Exception e) {
            handleRetryFailure(record, ack, e);
        }
    }
}
```

---

## Dead Letter Queue (DLQ)

### Spring Kafka DLQ

```java
@Configuration
public class DLQConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // Configure DLQ
        factory.setCommonErrorHandler(
            new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate(),
                    (record, ex) -> new TopicPartition(
                        record.topic() + ".dlq",
                        record.partition()
                    )
                ),
                new FixedBackOff(1000L, 3L)  // 3 retries with 1s backoff
            )
        );
        
        return factory;
    }
}

@Service
public class DLQConsumer {
    
    @KafkaListener(topics = "events.dlq")
    public void consumeDLQ(ConsumerRecord<String, FailedEvent> record) {
        FailedEvent failedEvent = record.value();
        
        // Log for investigation
        logger.error("DLQ Message - Topic: {}, Offset: {}, Error: {}",
            record.topic(),
            record.offset(),
            failedEvent.getError());
        
        // Store in database for manual review
        dlqRepository.save(failedEvent);
        
        // Alert operations team
        alertService.sendDLQAlert(failedEvent);
    }
}
```

### Vanilla Kafka DLQ

```java
public class VanillaDLQProducer {
    
    private final KafkaProducer<String, String> dlqProducer;
    
    public VanillaDLQProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        
        this.dlqProducer = new KafkaProducer<>(props);
    }
    
    public void sendToDLQ(
            ConsumerRecord<String, String> failedRecord,
            Exception error) {
        
        // Create DLQ message with metadata
        Map<String, String> headers = new HashMap<>();
        headers.put("original.topic", failedRecord.topic());
        headers.put("original.partition", String.valueOf(failedRecord.partition()));
        headers.put("original.offset", String.valueOf(failedRecord.offset()));
        headers.put("error.message", error.getMessage());
        headers.put("error.timestamp", Instant.now().toString());
        
        ProducerRecord<String, String> dlqRecord = 
            new ProducerRecord<>(
                failedRecord.topic() + ".dlq",
                failedRecord.key(),
                failedRecord.value()
            );
        
        headers.forEach((key, value) -> 
            dlqRecord.headers().add(key, value.getBytes())
        );
        
        try {
            dlqProducer.send(dlqRecord).get();
            logger.info("Sent to DLQ: {}", failedRecord.key());
        } catch (Exception e) {
            logger.error("Failed to send to DLQ", e);
            // Critical: Store locally if DLQ send fails
            storeLocally(failedRecord, error);
        }
    }
    
    private void storeLocally(ConsumerRecord<String, String> record, Exception error) {
        // Fallback: Write to local file/database
        try (FileWriter writer = new FileWriter("dlq-fallback.log", true)) {
            writer.write(String.format("%s|%s|%s|%s%n",
                Instant.now(),
                record.topic(),
                record.value(),
                error.getMessage()
            ));
        } catch (IOException e) {
            logger.error("Critical: Cannot write to fallback DLQ", e);
        }
    }
}
```

---

## Exactly-Once Semantics

### Consume-Transform-Produce Pattern

```java
/**
 * Exactly-Once Semantics (EOS)
 * 
 * Requirements:
 * - Transactional producer
 * - Transactional consumer (read_committed)
 * - Manual offset management
 * - Commit offsets within transaction
 */

// Spring Kafka EOS
@Configuration
public class ExactlyOnceConfig {
    
    @Bean
    public ProducerFactory<String, Event> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "eos-producer");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public ConsumerFactory<String, Event> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(config);
    }
}

@Service
public class ExactlyOnceConsumer {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    
    @KafkaListener(topics = "input-topic", containerFactory = "eosFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        kafkaTemplate.executeInTransaction(operations -> {
            try {
                // Transform
                Event transformed = transform(record.value());
                
                // Produce to output topic
                operations.send("output-topic", transformed);
                
                // Commit offset within transaction
                ack.acknowledge();
                
                return true;
            } catch (Exception e) {
                throw new RuntimeException("Processing failed", e);
            }
        });
    }
}

// Vanilla Kafka EOS
public class VanillaExactlyOnce {
    
    private final KafkaConsumer<String, String> consumer;
    private final KafkaProducer<String, String> producer;
    
    public VanillaExactlyOnce() {
        // Consumer with read_committed
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        this.consumer = new KafkaConsumer<>(consumerProps);
        
        // Transactional producer
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "eos-producer-1");
        producerProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        this.producer = new KafkaProducer<>(producerProps);
        producer.initTransactions();
        
        consumer.subscribe(Collections.singletonList("input-topic"));
    }
    
    public void processExactlyOnce() {
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(100));
            
            if (!records.isEmpty()) {
                try {
                    producer.beginTransaction();
                    
                    for (ConsumerRecord<String, String> record : records) {
                        // Transform
                        String transformed = transform(record.value());
                        
                        // Produce
                        producer.send(new ProducerRecord<>(
                            "output-topic",
                            record.key(),
                            transformed
                        ));
                    }
                    
                    // Commit offsets within transaction
                    Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                    for (ConsumerRecord<String, String> record : records) {
                        offsets.put(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1)
                        );
                    }
                    producer.sendOffsetsToTransaction(
                        offsets,
                        consumer.groupMetadata()
                    );
                    
                    producer.commitTransaction();
                    
                } catch (Exception e) {
                    producer.abortTransaction();
                    logger.error("Transaction aborted", e);
                }
            }
        }
    }
}
```

---

## Recovery Patterns

### Idempotent Processing

```java
@Service
public class IdempotentProcessor {
    
    private final ProcessedMessageRepository repository;
    
    @KafkaListener(topics = "events", containerFactory = "manualCommitFactory")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        String messageId = record.key();
        
        // Check if already processed (idempotency check)
        if (repository.existsById(messageId)) {
            logger.info("Message {} already processed, skipping", messageId);
            ack.acknowledge();
            return;
        }
        
        try {
            // Process message
            processEvent(record.value());
            
            // Mark as processed
            repository.save(new ProcessedMessage(
                messageId,
                record.topic(),
                record.partition(),
                record.offset(),
                Instant.now()
            ));
            
            // Commit offset
            ack.acknowledge();
            
        } catch (Exception e) {
            logger.error("Processing failed", e);
            // Don't commit - will be reprocessed
        }
    }
}
```

### Database + Kafka Transaction

```java
@Service
public class DatabaseKafkaTransaction {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    private final EntityManager entityManager;
    
    @Transactional  // Database transaction
    public void processWithDatabaseTransaction(Event event) {
        // Save to database (in transaction)
        EventEntity entity = new EventEntity(event);
        entityManager.persist(entity);
        
        // Send to Kafka (transactional)
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send("output-topic", event);
            return true;
        });
        
        // Both commit or both rollback
    }
}
```

### Outbox Pattern

```java
/**
 * Outbox Pattern - Guarantee message publishing
 * 
 * Flow:
 * 1. Write business data + outbox entry in same DB transaction
 * 2. Background job reads outbox and publishes to Kafka
 * 3. Mark as published after successful send
 */

@Service
public class OutboxPatternService {
    
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    
    @Transactional
    public void createOrder(Order order) {
        // Save order
        orderRepository.save(order);
        
        // Save outbox entry (same transaction)
        OutboxEvent outboxEvent = new OutboxEvent(
            UUID.randomUUID().toString(),
            "order-created",
            objectMapper.writeValueAsString(order),
            Instant.now(),
            false
        );
        outboxRepository.save(outboxEvent);
        
        // Both commit atomically
    }
}

@Service
public class OutboxPublisher {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 1000)  // Every second
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository
            .findByPublishedFalse();
        
        pending.forEach(event -> {
            try {
                kafkaTemplate.send(
                    event.getEventType(),
                    event.getPayload()
                ).get();  // Wait for ack
                
                // Mark as published
                event.setPublished(true);
                outboxRepository.save(event);
                
            } catch (Exception e) {
                logger.error("Failed to publish outbox event", e);
                // Will retry on next iteration
            }
        });
    }
}
```

---

## Monitoring and Alerting

### Lag Monitoring

```java
@Service
public class ZeroLossMonitoring {
    
    private final AdminClient adminClient;
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedDelay = 30000)  // Every 30 seconds
    public void monitorConsumerLag() {
        try {
            Map<TopicPartition, OffsetAndMetadata> committed = 
                adminClient.listConsumerGroupOffsets("my-group")
                    .partitionsToOffsetAndMetadata()
                    .get();
            
            Map<TopicPartition, Long> endOffsets = getEndOffsets(committed.keySet());
            
            committed.forEach((tp, offset) -> {
                long lag = endOffsets.get(tp) - offset.offset();
                
                meterRegistry.gauge(
                    "kafka.consumer.lag",
                    Tags.of("topic", tp.topic(), "partition", String.valueOf(tp.partition())),
                    lag
                );
                
                // Alert if lag > threshold
                if (lag > 10000) {
                    alertService.sendAlert(String.format(
                        "HIGH LAG: %d messages on %s-%d",
                        lag, tp.topic(), tp.partition()
                    ));
                }
            });
        } catch (Exception e) {
            logger.error("Failed to monitor lag", e);
        }
    }
    
    @Scheduled(fixedDelay = 60000)
    public void monitorDLQSize() {
        // Monitor DLQ topic size
        long dlqSize = getDLQMessageCount();
        
        if (dlqSize > 100) {
            alertService.sendAlert(String.format(
                "DLQ SIZE: %d messages in dead letter queue",
                dlqSize
            ));
        }
    }
}
```

---

## Best Practices

### ✅ DO

1. **Producer Settings**
   ```yaml
   acks: all
   enable.idempotence: true
   retries: Integer.MAX_VALUE
   max.in.flight.requests.per.connection: 5
   ```

2. **Consumer Settings**
   ```yaml
   enable.auto.commit: false
   isolation.level: read_committed
   auto.offset.reset: earliest
   ```

3. **Manual Offset Management**
   ```java
   processMessage(record);
   ack.acknowledge();  // Commit AFTER processing
   ```

4. **Implement DLQ**
5. **Use Transactions for EOS**
6. **Idempotent Processing**
7. **Monitor Lag Continuously**
8. **Test Failure Scenarios**
9. **Use Retry with Backoff**
10. **Alert on Anomalies**

### ❌ DON'T

1. **Don't auto-commit before processing**
2. **Don't ignore errors**
3. **Don't lose messages on DLQ send failure**
4. **Don't use acks=1 in production**
5. **Don't skip offset validation**

---

## Conclusion

**Zero Message Loss Checklist:**

```
Producer:
☑ acks=all
☑ enable.idempotence=true
☑ retries=Integer.MAX_VALUE
☑ Handle send failures

Broker:
☑ replication.factor >= 3
☑ min.insync.replicas >= 2
☑ unclean.leader.election.enable=false

Consumer:
☑ enable.auto.commit=false
☑ Manual offset management
☑ Commit after processing
☑ Error handling + retry
☑ Dead letter queue
☑ Idempotent processing

Monitoring:
☑ Consumer lag
☑ DLQ size
☑ Error rates
☑ Rebalances
```

**Remember**: Zero message loss requires careful configuration at every layer - producer, broker, and consumer!
