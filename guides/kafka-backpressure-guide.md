# Effective Kafka Backpressure Handling

A comprehensive guide to managing backpressure in Kafka consumers, preventing lag, and building resilient event-driven systems.

---

## Table of Contents

1. [Understanding Kafka Backpressure](#understanding-kafka-backpressure)
2. [Consumer Configuration](#consumer-configuration)
3. [Manual Offset Management](#manual-offset-management)
4. [Batch Processing](#batch-processing)
5. [Rate Limiting and Throttling](#rate-limiting-and-throttling)
6. [Pause and Resume](#pause-and-resume)
7. [Parallel Processing](#parallel-processing)
8. [Reactor Kafka](#reactor-kafka)
9. [Error Handling and Dead Letter Queues](#error-handling-and-dead-letter-queues)
10. [Monitoring and Metrics](#monitoring-and-metrics)
11. [Scaling Strategies](#scaling-strategies)
12. [Real-World Patterns](#real-world-patterns)

---

## Understanding Kafka Backpressure

### What is Backpressure in Kafka?

```java
/**
 * Backpressure occurs when:
 * - Consumer processes messages slower than producer sends them
 * - Consumer lag increases (offset gap grows)
 * - Processing time > poll interval
 * - Memory/CPU resources exhausted
 * 
 * Symptoms:
 * - Growing consumer lag
 * - Rebalancing loops
 * - Out of memory errors
 * - Missed SLA/timeouts
 * 
 * Solutions:
 * 1. Control poll rate (max.poll.records, max.poll.interval.ms)
 * 2. Pause/Resume consumption
 * 3. Batch processing with backpressure
 * 4. Parallel processing with limits
 * 5. Rate limiting
 * 6. Reactive streams (Reactor Kafka)
 */

// Problem: Fast producer, slow consumer
// Producer: 10,000 msg/sec
// Consumer: 100 msg/sec
// Result: Lag grows by 9,900 msg/sec!
```

---

## Consumer Configuration

### Key Configuration Properties

```java
/**
 * Critical Configuration for Backpressure Management:
 */

// application.yml
spring:
  kafka:
    consumer:
      # Max records per poll - CRITICAL for backpressure
      max-poll-records: 100  # Default: 500
      
      # Time between polls before rebalance
      max-poll-interval-ms: 300000  # 5 minutes
      
      # Session timeout
      session-timeout-ms: 10000  # 10 seconds
      
      # Heartbeat interval
      heartbeat-interval-ms: 3000  # 3 seconds
      
      # Auto commit - disable for manual control
      enable-auto-commit: false
      
      # Fetch settings
      fetch-min-size: 1
      fetch-max-wait-ms: 500
      
      properties:
        # Partition assignment strategy
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### Configuration Best Practices

```java
@Configuration
public class KafkaConsumerConfig {
    
    /**
     * Low throughput, high processing time
     * - Small max-poll-records (10-50)
     * - Large max-poll-interval-ms (5-10 min)
     */
    @Bean
    public ConsumerFactory<String, Event> slowConsumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_CONFIG_MS, 600000); // 10 min
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    /**
     * High throughput, fast processing
     * - Large max-poll-records (500-1000)
     * - Moderate max-poll-interval-ms (1-2 min)
     */
    @Bean
    public ConsumerFactory<String, Event> fastConsumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_CONFIG_MS, 120000); // 2 min
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

---

## Manual Offset Management

### Commit After Processing

```java
@Service
public class ManualCommitConsumer {
    
    @KafkaListener(
        topics = "orders",
        containerFactory = "manualCommitContainerFactory"
    )
    public void consume(
            ConsumerRecord<String, Order> record,
            Acknowledgment ack) {
        
        try {
            // Process message
            processOrder(record.value());
            
            // Commit offset AFTER successful processing
            ack.acknowledge();
            
        } catch (Exception e) {
            // Don't commit - message will be reprocessed
            logger.error("Processing failed", e);
            // Optionally: send to DLQ, retry later
        }
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> 
            manualCommitContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties()
            .setAckMode(AckMode.MANUAL);  // Manual commit
        
        return factory;
    }
}
```

### Batch Manual Commit

```java
@Service
public class BatchManualCommitConsumer {
    
    @KafkaListener(
        topics = "events",
        containerFactory = "batchManualContainerFactory"
    )
    public void consumeBatch(
            List<ConsumerRecord<String, Event>> records,
            Acknowledgment ack) {
        
        try {
            // Process entire batch
            List<Event> events = records.stream()
                .map(ConsumerRecord::value)
                .collect(Collectors.toList());
            
            processEventBatch(events);
            
            // Commit entire batch
            ack.acknowledge();
            
        } catch (Exception e) {
            logger.error("Batch processing failed", e);
            // Entire batch will be reprocessed
        }
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            batchManualContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        return factory;
    }
}
```

---

## Batch Processing

### Controlled Batch Size

```java
@Service
public class BatchProcessor {
    
    private final List<Event> buffer = new ArrayList<>();
    private final Lock lock = new ReentrantLock();
    private static final int BATCH_SIZE = 100;
    private static final Duration BATCH_TIMEOUT = Duration.ofSeconds(5);
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        lock.lock();
        try {
            buffer.add(record.value());
            
            // Process when batch size reached
            if (buffer.size() >= BATCH_SIZE) {
                processBatch();
                ack.acknowledge();
            }
        } finally {
            lock.unlock();
        }
    }
    
    @Scheduled(fixedDelay = 5000)  // Flush every 5 seconds
    public void flushBuffer() {
        lock.lock();
        try {
            if (!buffer.isEmpty()) {
                processBatch();
            }
        } finally {
            lock.unlock();
        }
    }
    
    private void processBatch() {
        try {
            // Batch insert to database
            eventRepository.saveAll(buffer);
            logger.info("Processed batch of {} events", buffer.size());
        } finally {
            buffer.clear();
        }
    }
}
```

### Spring Kafka Batch Listener

```java
@Service
public class SpringBatchListener {
    
    @KafkaListener(
        topics = "orders",
        containerFactory = "batchListenerFactory"
    )
    public void consumeBatch(
            List<ConsumerRecord<String, Order>> records,
            Acknowledgment ack) {
        
        logger.info("Processing batch of {} records", records.size());
        
        // Extract values
        List<Order> orders = records.stream()
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());
        
        // Batch processing
        orderService.processBatch(orders);
        
        // Commit entire batch
        ack.acknowledge();
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Order> 
            batchListenerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Order> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // Batch size control
        factory.getContainerProperties().setPollTimeout(3000);
        
        return factory;
    }
}
```

---

## Rate Limiting and Throttling

### Semaphore-Based Rate Limiting

```java
@Service
public class RateLimitedConsumer {
    
    // Limit to 100 concurrent processing
    private final Semaphore semaphore = new Semaphore(100);
    private final ExecutorService executor = Executors.newFixedThreadPool(100);
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            // Acquire permit (blocks if limit reached)
            semaphore.acquire();
            
            // Process asynchronously
            executor.submit(() -> {
                try {
                    processEvent(record.value());
                    ack.acknowledge();
                } catch (Exception e) {
                    logger.error("Processing failed", e);
                } finally {
                    semaphore.release();
                }
            });
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    @PreDestroy
    public void shutdown() {
        executor.shutdown();
    }
}
```

### Guava RateLimiter

```java
@Service
public class GuavaRateLimiter {
    
    // Limit to 1000 messages per second
    private final RateLimiter rateLimiter = RateLimiter.create(1000.0);
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        // Wait for permit (throttles consumption)
        rateLimiter.acquire();
        
        try {
            processEvent(record.value());
            ack.acknowledge();
        } catch (Exception e) {
            logger.error("Processing failed", e);
        }
    }
}
```

### Resilience4j Rate Limiter

```java
@Service
public class Resilience4jRateLimiter {
    
    private final io.github.resilience4j.ratelimiter.RateLimiter rateLimiter;
    
    public Resilience4jRateLimiter() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(1000)  // 1000 requests
            .limitRefreshPeriod(Duration.ofSeconds(1))  // per second
            .timeoutDuration(Duration.ofSeconds(5))
            .build();
        
        this.rateLimiter = io.github.resilience4j.ratelimiter.RateLimiter
            .of("kafka-consumer", config);
    }
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            // Acquire permit with timeout
            if (RateLimiter.waitForPermission(rateLimiter, 1)) {
                processEvent(record.value());
                ack.acknowledge();
            } else {
                logger.warn("Rate limit timeout");
            }
        } catch (Exception e) {
            logger.error("Processing failed", e);
        }
    }
}
```

---

## Pause and Resume

### Dynamic Pause/Resume Based on Load

```java
@Service
public class PauseResumeConsumer {
    
    private final KafkaListenerEndpointRegistry registry;
    private final Semaphore semaphore = new Semaphore(100);
    
    @KafkaListener(id = "pausable-consumer", topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            semaphore.acquire();
            
            // Check if overwhelmed
            if (semaphore.availablePermits() < 10) {
                pauseConsumer();
            }
            
            processEvent(record.value());
            ack.acknowledge();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();
            
            // Resume if capacity available
            if (semaphore.availablePermits() > 50) {
                resumeConsumer();
            }
        }
    }
    
    private void pauseConsumer() {
        MessageListenerContainer container = registry
            .getListenerContainer("pausable-consumer");
        
        if (container != null && !container.isPaused()) {
            container.pause();
            logger.warn("Consumer paused due to high load");
        }
    }
    
    private void resumeConsumer() {
        MessageListenerContainer container = registry
            .getListenerContainer("pausable-consumer");
        
        if (container != null && container.isPaused()) {
            container.resume();
            logger.info("Consumer resumed");
        }
    }
}
```

### Pause on Error

```java
@Service
public class ErrorBasedPauseConsumer {
    
    private final KafkaListenerEndpointRegistry registry;
    private final AtomicInteger errorCount = new AtomicInteger(0);
    private static final int ERROR_THRESHOLD = 10;
    
    @KafkaListener(id = "error-pause-consumer", topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            processEvent(record.value());
            ack.acknowledge();
            
            // Reset error count on success
            errorCount.set(0);
            
        } catch (Exception e) {
            logger.error("Processing failed", e);
            
            // Pause if too many errors
            if (errorCount.incrementAndGet() >= ERROR_THRESHOLD) {
                pauseConsumer();
                scheduleResume();
            }
        }
    }
    
    private void pauseConsumer() {
        MessageListenerContainer container = registry
            .getListenerContainer("error-pause-consumer");
        
        if (container != null) {
            container.pause();
            logger.warn("Consumer paused due to {} errors", errorCount.get());
        }
    }
    
    private void scheduleResume() {
        // Resume after 1 minute
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
        scheduler.schedule(() -> {
            MessageListenerContainer container = registry
                .getListenerContainer("error-pause-consumer");
            
            if (container != null) {
                errorCount.set(0);
                container.resume();
                logger.info("Consumer resumed after cooldown");
            }
        }, 1, TimeUnit.MINUTES);
    }
}
```

---

## Parallel Processing

### ThreadPoolTaskExecutor

```java
@Configuration
public class ParallelConsumerConfig {
    
    @Bean
    public ThreadPoolTaskExecutor kafkaExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("kafka-consumer-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            parallelListenerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // Concurrency
        factory.setConcurrency(3);  // 3 consumer threads per partition
        
        return factory;
    }
}

@Service
public class ParallelConsumer {
    
    private final ThreadPoolTaskExecutor executor;
    
    @KafkaListener(
        topics = "events",
        containerFactory = "parallelListenerFactory"
    )
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        executor.submit(() -> {
            try {
                processEvent(record.value());
                ack.acknowledge();
            } catch (Exception e) {
                logger.error("Processing failed", e);
            }
        });
    }
}
```

### Per-Partition Parallel Processing

```java
@Service
public class PartitionParallelConsumer {
    
    private final Map<Integer, ExecutorService> executorsByPartition = 
        new ConcurrentHashMap<>();
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        int partition = record.partition();
        
        // One executor per partition (maintains order within partition)
        ExecutorService executor = executorsByPartition.computeIfAbsent(
            partition,
            p -> Executors.newSingleThreadExecutor()
        );
        
        executor.submit(() -> {
            try {
                processEvent(record.value());
                ack.acknowledge();
            } catch (Exception e) {
                logger.error("Processing failed", e);
            }
        });
    }
    
    @PreDestroy
    public void shutdown() {
        executorsByPartition.values().forEach(ExecutorService::shutdown);
    }
}
```

---

## Reactor Kafka

### Reactive Kafka Consumer

```java
import reactor.kafka.receiver.KafkaReceiver;
import reactor.kafka.receiver.ReceiverOptions;

@Configuration
public class ReactorKafkaConfig {
    
    @Bean
    public ReceiverOptions<String, Event> receiverOptions() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "reactor-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        return ReceiverOptions.<String, Event>create(props)
            .subscription(Collections.singleton("events"));
    }
    
    @Bean
    public KafkaReceiver<String, Event> kafkaReceiver(
            ReceiverOptions<String, Event> receiverOptions) {
        return KafkaReceiver.create(receiverOptions);
    }
}
```

### Backpressure with Reactor

```java
@Service
public class ReactiveKafkaConsumer {
    
    private final KafkaReceiver<String, Event> receiver;
    
    @PostConstruct
    public void startConsuming() {
        receiver.receive()
            // Automatic backpressure handling
            .limitRate(100)  // Request 100 at a time
            
            // Process with controlled concurrency
            .flatMap(record -> 
                processEvent(record.value())
                    .doOnSuccess(v -> record.receiverOffset().acknowledge())
                    .onErrorResume(e -> {
                        logger.error("Processing failed", e);
                        return Mono.empty();
                    }),
                10  // Max 10 concurrent processing
            )
            .subscribe();
    }
    
    private Mono<Void> processEvent(Event event) {
        return Mono.fromCallable(() -> {
            // Process event
            eventService.process(event);
            return null;
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

### Advanced Reactor Kafka Patterns

```java
@Service
public class AdvancedReactorConsumer {
    
    private final KafkaReceiver<String, Event> receiver;
    
    @PostConstruct
    public void startConsuming() {
        receiver.receive()
            // Batching
            .buffer(100)  // Group into batches of 100
            .flatMap(batch -> processBatch(batch))
            
            // Or time-based batching
            .bufferTimeout(100, Duration.ofSeconds(5))
            .flatMap(batch -> processBatch(batch))
            
            // With retry
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))
            
            // Rate limiting
            .onBackpressureBuffer(1000)
            
            .subscribe();
    }
    
    private Mono<Void> processBatch(List<ReceiverRecord<String, Event>> batch) {
        return Mono.fromRunnable(() -> {
            List<Event> events = batch.stream()
                .map(ReceiverRecord::value)
                .collect(Collectors.toList());
            
            eventService.processBatch(events);
            
            // Commit batch
            batch.forEach(record -> 
                record.receiverOffset().acknowledge()
            );
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

---

## Error Handling and Dead Letter Queues

### Retry with DLQ

```java
@Service
public class RetryAndDLQConsumer {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    private static final int MAX_RETRIES = 3;
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        Event event = record.value();
        int retryCount = getRetryCount(event);
        
        try {
            processEvent(event);
            ack.acknowledge();
            
        } catch (Exception e) {
            if (retryCount < MAX_RETRIES) {
                // Send to retry topic with incremented counter
                event.setRetryCount(retryCount + 1);
                kafkaTemplate.send("events.retry", event);
                ack.acknowledge();
                
                logger.warn("Sent to retry topic, attempt {}", retryCount + 1);
            } else {
                // Max retries exceeded - send to DLQ
                kafkaTemplate.send("events.dlq", event);
                ack.acknowledge();
                
                logger.error("Max retries exceeded, sent to DLQ", e);
            }
        }
    }
    
    private int getRetryCount(Event event) {
        return event.getRetryCount() != null ? event.getRetryCount() : 0;
    }
}
```

### Exponential Backoff Retry

```java
@Service
public class ExponentialBackoffRetry {
    
    private final KafkaTemplate<String, Event> kafkaTemplate;
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        Event event = record.value();
        
        try {
            processEvent(event);
            ack.acknowledge();
            
        } catch (Exception e) {
            int retryCount = getRetryCount(event);
            
            if (retryCount < 5) {
                // Calculate delay: 2^retryCount seconds
                long delaySeconds = (long) Math.pow(2, retryCount);
                
                // Send to delayed retry topic
                event.setRetryCount(retryCount + 1);
                event.setRetryAfter(Instant.now().plusSeconds(delaySeconds));
                
                kafkaTemplate.send("events.retry." + delaySeconds + "s", event);
                ack.acknowledge();
                
                logger.warn("Retry scheduled after {} seconds", delaySeconds);
            } else {
                sendToDLQ(event, e);
                ack.acknowledge();
            }
        }
    }
    
    private void sendToDLQ(Event event, Exception e) {
        event.setError(e.getMessage());
        event.setFailedAt(Instant.now());
        kafkaTemplate.send("events.dlq", event);
    }
}
```

---

## Monitoring and Metrics

### Consumer Lag Monitoring

```java
@Service
public class ConsumerLagMonitor {
    
    private final AdminClient adminClient;
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedDelay = 60000)  // Every minute
    public void monitorLag() {
        try {
            Map<TopicPartition, OffsetAndMetadata> committed = 
                adminClient.listConsumerGroupOffsets("my-group")
                    .partitionsToOffsetAndMetadata()
                    .get();
            
            Map<TopicPartition, Long> endOffsets = 
                adminClient.listOffsets(
                    committed.keySet().stream()
                        .collect(Collectors.toMap(
                            tp -> tp,
                            tp -> OffsetSpec.latest()
                        ))
                ).all().get().entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> e.getValue().offset()
                ));
            
            committed.forEach((tp, offset) -> {
                long lag = endOffsets.get(tp) - offset.offset();
                
                meterRegistry.gauge(
                    "kafka.consumer.lag",
                    Tags.of(
                        "topic", tp.topic(),
                        "partition", String.valueOf(tp.partition())
                    ),
                    lag
                );
                
                if (lag > 10000) {
                    logger.warn("High lag detected: {} messages on {}", lag, tp);
                }
            });
            
        } catch (Exception e) {
            logger.error("Failed to monitor lag", e);
        }
    }
}
```

### Processing Metrics

```java
@Service
public class MetricsConsumer {
    
    private final MeterRegistry meterRegistry;
    private final Counter processedCounter;
    private final Counter failedCounter;
    private final Timer processingTimer;
    
    public MetricsConsumer(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.processedCounter = Counter.builder("kafka.messages.processed")
            .tag("topic", "events")
            .register(meterRegistry);
        
        this.failedCounter = Counter.builder("kafka.messages.failed")
            .tag("topic", "events")
            .register(meterRegistry);
        
        this.processingTimer = Timer.builder("kafka.processing.time")
            .tag("topic", "events")
            .register(meterRegistry);
    }
    
    @KafkaListener(topics = "events")
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            processEvent(record.value());
            ack.acknowledge();
            
            processedCounter.increment();
            sample.stop(processingTimer);
            
        } catch (Exception e) {
            failedCounter.increment();
            logger.error("Processing failed", e);
        }
    }
}
```

---

## Scaling Strategies

### Horizontal Scaling

```java
/**
 * Increase Partitions and Consumers:
 * 
 * 1. Add more partitions to topic
 * 2. Deploy more consumer instances
 * 3. Each partition consumed by one consumer in group
 * 4. Max consumers = number of partitions
 */

// Scale up: 3 partitions -> 10 partitions
kafka-topics --alter --topic events --partitions 10

// Deploy 10 consumer instances
// Each gets 1 partition
```

### Vertical Scaling - Concurrency

```java
@Configuration
public class ConcurrentConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            concurrentFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        
        // 10 consumer threads per instance
        factory.setConcurrency(10);
        
        return factory;
    }
}
```

---

## Real-World Patterns

### High-Throughput Pattern

```java
@Service
public class HighThroughputConsumer {
    
    @KafkaListener(
        topics = "high-volume-events",
        containerFactory = "highThroughputFactory"
    )
    public void consumeBatch(
            List<ConsumerRecord<String, Event>> records,
            Acknowledgment ack) {
        
        // Batch process
        List<Event> events = records.stream()
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());
        
        // Bulk database insert
        eventRepository.batchInsert(events);
        
        // Single commit for entire batch
        ack.acknowledge();
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            highThroughputFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // High throughput settings
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 1000);
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024 * 1024);  // 1 MB
        
        return factory;
    }
}
```

### Critical Processing Pattern

```java
@Service
public class CriticalProcessingConsumer {
    
    private final Semaphore semaphore = new Semaphore(10);  // Max 10 concurrent
    
    @KafkaListener(
        topics = "critical-events",
        containerFactory = "criticalFactory"
    )
    public void consume(ConsumerRecord<String, Event> record, Acknowledgment ack) {
        try {
            // Control concurrency
            semaphore.acquire();
            
            // Process with retries
            processWithRetry(record.value());
            
            // Commit only after success
            ack.acknowledge();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (Exception e) {
            // Send to DLQ after retries
            sendToDLQ(record.value(), e);
            ack.acknowledge();
        } finally {
            semaphore.release();
        }
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Event> 
            criticalFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Event> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        
        // Low throughput, high reliability
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 600000);  // 10 min
        
        return factory;
    }
}
```

---

## Best Practices

### ✅ DO

1. **Use manual offset management**
   ```java
   factory.getContainerProperties().setAckMode(AckMode.MANUAL);
   ack.acknowledge();  // Commit after processing
   ```

2. **Configure max.poll.records based on processing time**
   ```java
   // Slow processing: small batch
   props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);
   
   // Fast processing: large batch
   props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
   ```

3. **Use batch processing for high throughput**
4. **Implement rate limiting for downstream services**
5. **Pause/Resume based on load**
6. **Monitor consumer lag continuously**
7. **Use DLQ for failed messages**
8. **Scale partitions with load**
9. **Use Reactor Kafka for reactive apps**
10. **Set appropriate timeouts**

### ❌ DON'T

1. **Don't block for long periods**
2. **Don't auto-commit without processing**
3. **Don't ignore consumer lag**
4. **Don't use unlimited concurrency**
5. **Don't process without error handling**

---

## Conclusion

**Key Takeaways:**

1. **Control poll rate** - max.poll.records, max.poll.interval.ms
2. **Manual commits** - commit after successful processing
3. **Batch processing** - increase throughput
4. **Rate limiting** - protect downstream
5. **Pause/Resume** - dynamic backpressure
6. **Parallel processing** - controlled concurrency
7. **Reactor Kafka** - reactive backpressure
8. **Monitor lag** - detect issues early
9. **DLQ pattern** - handle failures
10. **Scale appropriately** - partitions + consumers

**Remember**: Backpressure is about matching consumption rate to processing capacity!
