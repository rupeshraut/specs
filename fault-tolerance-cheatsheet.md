# 🛡️ Distributed Systems Fault Tolerance Cheat Sheet

> **Purpose:** Production-ready resilience patterns for distributed systems. Reference before designing any inter-service communication, external integration, or async processing.
> **Stack context:** Java 21+ / Spring Boot 3.x / Resilience4j / Kafka / MongoDB / ActiveMQ Artemis

---

## 📋 The Fault Tolerance Decision Framework

Before adding any resilience pattern, answer:

| Question | Determines |
|----------|-----------|
| Is the failure **transient** or **permanent**? | Retry vs. fail fast |
| Is the operation **idempotent**? | Safe to retry or not |
| What's the **blast radius** if this fails? | Circuit breaker scope |
| Is there a **meaningful fallback**? | Fallback vs. propagate error |
| What's the **acceptable latency** budget? | Timeout value |
| Can the caller **wait** or must it **proceed**? | Sync vs. async resilience |
| Is **ordering** important? | Retry queue strategy |

---

## 🔄 Pattern 1: Retry

> **When:** Transient failures (network blip, temporary unavailability, optimistic lock conflict)
> **Never when:** The operation is NOT idempotent, or failure is clearly permanent (400, 404, validation error)

### Retry Decision Tree

```
Is the error transient?
├── NO → Don't retry. Fail fast or dead-letter.
└── YES → Is the operation idempotent?
    ├── NO → Don't retry (or ensure idempotency first via idempotency key)
    └── YES → What type of backoff?
        ├── Fixed delay     → Equal intervals (simple, good for rate limits)
        ├── Exponential     → Doubling intervals (standard for most cases)
        └── Exponential     → With jitter (BEST for distributed systems — prevents thundering herd)
            + jitter
```

### Resilience4j Retry Configuration

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        var config = io.github.resilience4j.retry.RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                500,       // initialInterval ms
                2.0,       // multiplier
                0.5        // randomizationFactor (adds ±50% jitter)
            ))
            .retryOnException(this::isRetryable)
            .ignoreExceptions(
                ValidationException.class,
                BusinessRuleException.class,
                DuplicatePaymentException.class
            )
            .failAfterMaxAttempts(true)
            .build();

        return RetryRegistry.of(config);
    }

    private boolean isRetryable(Throwable t) {
        if (t instanceof HttpClientErrorException http) {
            // Never retry 4xx (client errors) except 429 (rate limited)
            return http.getStatusCode().value() == 429;
        }
        // Retry on connection errors, timeouts, 5xx
        return t instanceof ConnectException
            || t instanceof SocketTimeoutException
            || t instanceof HttpServerErrorException
            || t instanceof MongoTimeoutException
            || t instanceof JMSException;
    }
}
```

### Retry Classification Matrix

| HTTP Status / Error | Retry? | Reason |
|---------------------|--------|--------|
| **400** Bad Request | ❌ Never | Permanent — fix the request |
| **401** Unauthorized | ❌ Never | Auth issue — retry won't help |
| **403** Forbidden | ❌ Never | Permission issue |
| **404** Not Found | ❌ Never | Resource doesn't exist |
| **409** Conflict | ⚠️ Maybe | Retry if using optimistic locking |
| **422** Unprocessable | ❌ Never | Business rule violation |
| **429** Too Many Requests | ✅ Yes | Respect `Retry-After` header |
| **500** Internal Server Error | ✅ Yes | Transient server issue |
| **502** Bad Gateway | ✅ Yes | Upstream transient failure |
| **503** Service Unavailable | ✅ Yes | Temporary overload |
| **504** Gateway Timeout | ✅ Yes | Upstream timeout |
| **Connection refused** | ✅ Yes | Service restarting |
| **Socket timeout** | ✅ Yes | Network latency spike |
| **SSL handshake failure** | ❌ Never | Config issue |
| **DNS resolution failure** | ⚠️ Maybe | Usually transient, but short retry window |
| **MongoDB duplicate key** | ❌ Never | Permanent — data conflict |
| **MongoDB timeout** | ✅ Yes | Transient |
| **Kafka producer timeout** | ✅ Yes | Broker overloaded |
| **JMS connection lost** | ✅ Yes | Broker failover in progress |

### Retry with Idempotency Key Pattern

```java
public record PaymentRequest(
    String idempotencyKey,  // Client-generated UUID
    Money amount,
    String fromAccount,
    String toAccount
) {}

@Component
public class IdempotentPaymentService {

    private final MongoTemplate mongoTemplate;

    public PaymentResult processPayment(PaymentRequest request) {
        // Check if already processed
        var existing = mongoTemplate.findById(
            request.idempotencyKey(), IdempotencyRecord.class);

        if (existing != null) {
            return existing.result(); // Return cached result — safe to "retry"
        }

        // Process and store atomically
        PaymentResult result = doProcess(request);

        mongoTemplate.insert(new IdempotencyRecord(
            request.idempotencyKey(),
            result,
            Instant.now(),
            Instant.now().plus(Duration.ofHours(24)) // TTL
        ));

        return result;
    }
}
```

---

## 🔌 Pattern 2: Circuit Breaker

> **When:** Protecting your system from repeatedly calling a failing downstream service.
> **Mental model:** Like an electrical circuit breaker — trips open to prevent damage, then cautiously tests if service is back.

### Circuit Breaker States

```
                success
    ┌──────────────────────────┐
    ▼                          │
 ┌──────┐   failure threshold  ┌──────────┐   wait duration   ┌───────────┐
 │CLOSED │ ──────────────────► │   OPEN   │ ────────────────► │ HALF-OPEN │
 └──────┘   (e.g., 50% of     └──────────┘   (e.g., 30s)     └───────────┘
    ▲        last 10 calls)     │                               │    │
    │                           │ all calls fail fast            │    │
    │                           │ (no downstream call)           │    │
    │                           ▼                                │    │
    │                      FallbackResponse                      │    │
    │                                                            │    │
    │         success (permitted calls succeed)                  │    │
    └────────────────────────────────────────────────────────────┘    │
                                                                      │
              failure (permitted calls fail)                           │
              ─────────────────────────────────────────► OPEN ◄───────┘
```

### Resilience4j Circuit Breaker Configuration

```java
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        var config = CircuitBreakerConfig.custom()
            // Sliding window to measure failure rate
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)                        // evaluate last 10 calls
            .minimumNumberOfCalls(5)                      // need at least 5 calls before tripping

            // When to trip OPEN
            .failureRateThreshold(50)                     // open if ≥50% fail
            .slowCallRateThreshold(80)                    // open if ≥80% are slow
            .slowCallDurationThreshold(Duration.ofSeconds(3))

            // How long to stay OPEN before trying HALF-OPEN
            .waitDurationInOpenState(Duration.ofSeconds(30))

            // HALF-OPEN behavior
            .permittedNumberOfCallsInHalfOpenState(3)     // allow 3 test calls
            .automaticTransitionFromOpenToHalfOpenEnabled(true)

            // What counts as failure
            .recordExceptions(
                ConnectException.class,
                SocketTimeoutException.class,
                HttpServerErrorException.class
            )
            .ignoreExceptions(
                ValidationException.class,
                BusinessRuleException.class
            )
            .build();

        return CircuitBreakerRegistry.of(config);
    }
}
```

### Per-Service Circuit Breakers (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-gateway:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        sliding-window-size: 10
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
      fraud-service:
        failure-rate-threshold: 60
        sliding-window-size: 20
        wait-duration-in-open-state: 60s
      account-service:
        failure-rate-threshold: 40      # More sensitive — critical path
        sliding-window-size: 5
        wait-duration-in-open-state: 15s
```

### Circuit Breaker Scope Rules

```
❌ ONE circuit breaker for ALL downstream services
   → One slow service trips the breaker for everything

✅ ONE circuit breaker PER downstream service
   → Isolates failures to the service that's actually failing

✅ SEPARATE circuit breakers PER operation on the same service (if needed)
   → /payments might fail while /refunds is fine
```

---

## ⏱️ Pattern 3: Timeout

> **When:** ALWAYS. Every external call MUST have a timeout. No exceptions.
> **Rule:** Timeout is the FIRST resilience pattern you add — everything else is optional, timeout is mandatory.

### Timeout Budget Pattern

```
Total request budget: 5 seconds

Client ──► API Gateway ──► Payment Service ──► Payment Gateway
           (500ms)         (internal: 200ms)   (3000ms)
                           ──► Fraud Service
                               (1500ms)
                           ──► Account Service
                               (1000ms)

Budget consumed:
  Gateway overhead:    500ms
  Sequential calls:    3000ms (gateway) + 200ms (internal)
  Parallel calls:      max(1500ms, 1000ms) = 1500ms
  ─────────────────────────────
  Total:               ~5200ms → EXCEEDS BUDGET

Fix: Run fraud + account checks IN PARALLEL with structured concurrency
  Revised:             500ms + 200ms + max(1500ms, 3000ms) = 3700ms ✅
```

### Resilience4j Timeout Configuration

```java
@Bean
public TimeLimiterRegistry timeLimiterRegistry() {
    var config = TimeLimiterConfig.custom()
        .timeoutDuration(Duration.ofSeconds(3))
        .cancelRunningFuture(true)    // Cancel the underlying call on timeout
        .build();

    return TimeLimiterRegistry.of(config);
}
```

### Timeout Values — Decision Guide

| Call Type | Suggested Timeout | Reasoning |
|-----------|-------------------|-----------|
| **Internal microservice REST** | 1-3s | Should be fast; if not, something's wrong |
| **External payment gateway** | 5-10s | Third party, unpredictable |
| **Database query** | 2-5s | If longer, optimize the query |
| **MongoDB aggregation** | 5-15s | Complex pipelines can be slow |
| **Kafka produce** | 5-30s | Depends on acks config and broker load |
| **JMS send** | 3-10s | Broker should be responsive |
| **DNS resolution** | 2s | Should be cached; longer = problem |
| **SSL handshake** | 3s | Includes certificate validation |
| **File/S3 upload** | 30-60s | Size dependent |
| **Health check** | 1-2s | Must be fast by definition |

### HTTP Client Timeout Layers

```java
// ✅ Set ALL timeout layers — missing any one creates an unbounded wait
RestClient restClient = RestClient.builder()
    .requestFactory(new JdkClientHttpRequestFactory(
        HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2))   // TCP connection timeout
            .build()))
    .defaultHeaders(headers -> {
        headers.set("X-Request-Timeout", "3000");     // Inform downstream of budget
    })
    .build();

// Spring WebClient (reactive)
WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofSeconds(3))           // Response timeout
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000) // Connection timeout
    ))
    .build();
```

---

## 🚧 Pattern 4: Bulkhead

> **When:** Isolating resource consumption so one slow consumer doesn't starve others.
> **Mental model:** Ship bulkheads — one flooded compartment doesn't sink the whole ship.

### Bulkhead Types

```
Semaphore Bulkhead (Default)
─────────────────────────────
Limits concurrent calls. Lightweight, no thread pool.
Best for: Most use cases, virtual threads

Thread Pool Bulkhead
─────────────────────────────
Dedicated thread pool per downstream. Full isolation.
Best for: When you need hard isolation with platform threads
Note: Less relevant with virtual threads (use semaphore instead)
```

### Resilience4j Bulkhead Configuration

```java
// Semaphore bulkhead — limits concurrency
@Bean
public BulkheadRegistry bulkheadRegistry() {
    var config = BulkheadConfig.custom()
        .maxConcurrentCalls(25)              // max 25 concurrent calls to this downstream
        .maxWaitDuration(Duration.ofMillis(500)) // wait up to 500ms if all slots full
        .build();

    return BulkheadRegistry.of(config);
}
```

### Bulkhead Sizing Guide

```
Formula:
  maxConcurrentCalls = (requests_per_second) × (avg_latency_seconds) × safety_factor

Example:
  Payment gateway: 100 RPS × 0.5s avg latency × 1.5 safety = 75 concurrent calls

Per-service isolation:
  payment-gateway:    maxConcurrentCalls = 75
  fraud-service:      maxConcurrentCalls = 50
  account-service:    maxConcurrentCalls = 100
  notification:       maxConcurrentCalls = 25   (non-critical, lower priority)
```

---

## 🔀 Pattern 5: Fallback

> **When:** You can provide a degraded but acceptable response when the primary path fails.
> **Never when:** Data accuracy is critical (financial calculations, compliance checks).

### Fallback Strategy Matrix

| Scenario | Fallback Strategy | Example |
|----------|-------------------|---------|
| **Cache available** | Return stale data | Exchange rate from 5 min ago |
| **Default exists** | Return safe default | Default shipping rate |
| **Async possible** | Queue for later | Notification → dead-letter for retry |
| **Partial data OK** | Return what you have | Product page without reviews |
| **Nothing meaningful** | Fail with clear error | Payment → no fallback, fail clearly |

### Fallback Implementation Patterns

```java
@Component
public class ExchangeRateService {

    private final ExchangeRateClient client;
    private final CacheManager cacheManager;

    // Annotation-based with Resilience4j
    @CircuitBreaker(name = "exchange-rate", fallbackMethod = "getCachedRate")
    @Retry(name = "exchange-rate")
    @TimeLimiter(name = "exchange-rate")
    public Mono<BigDecimal> getRate(String from, String to) {
        return client.fetchRate(from, to);
    }

    // Fallback — MUST have same return type + Throwable parameter
    private Mono<BigDecimal> getCachedRate(String from, String to, Throwable t) {
        log.warn("Exchange rate service unavailable, using cached rate. Error: {}",
                 t.getMessage());

        var cached = cacheManager.getCache("exchange-rates")
            .get(from + "-" + to, BigDecimal.class);

        if (cached != null) {
            return Mono.just(cached);
        }

        // No cache available — fail explicitly, don't guess
        return Mono.error(new ServiceUnavailableException(
            "Exchange rate unavailable and no cached value exists"));
    }
}
```

### Fallback Anti-Patterns

```
❌ Silent fallback — returning a default without logging or metrics
   → You won't know the primary path has been failing for hours

❌ Fallback that calls another fragile service
   → Cascading failure moves to the fallback path

❌ Fallback for financial calculations
   → "Approximate" payment amounts create reconciliation nightmares

❌ Infinite fallback chains
   → Primary → Fallback A → Fallback B → ... → eventually fails anyway

✅ Fallback with observability
   → Log, emit metric, alert if fallback rate exceeds threshold
```

---

## 🔗 Pattern 6: Resilience Pattern Composition

> **Critical:** The ORDER of pattern decoration matters. Outer pattern wraps inner pattern.

### Recommended Composition Order

```
Request
  │
  ▼
┌─────────────┐
│   Retry     │  ← Outermost: retries the ENTIRE decorated call
├─────────────┤
│ Circuit     │  ← Records success/failure AFTER timeout check
│  Breaker    │
├─────────────┤
│  Bulkhead   │  ← Limits concurrency before making the call
├─────────────┤
│  Timeout    │  ← Innermost: directly wraps the actual call
├─────────────┤
│  Actual     │
│  Service    │
│  Call       │
└─────────────┘

Execution order (inside out):
  Timeout → Bulkhead → Circuit Breaker → Retry → Fallback

Why this order:
  1. Timeout: ensures the call doesn't hang
  2. Bulkhead: ensures we don't overwhelm the downstream
  3. Circuit Breaker: records the outcome (including timeouts)
  4. Retry: retries the whole chain if it fails
  5. Fallback: activates only after all retries are exhausted
```

### Spring Boot Annotation Composition

```java
@Component
public class PaymentGatewayClient {

    // Annotations are applied in the order: Retry → CircuitBreaker → Bulkhead → TimeLimiter
    // (Resilience4j Spring Boot handles the ordering)

    @Retry(name = "payment-gateway")
    @CircuitBreaker(name = "payment-gateway", fallbackMethod = "fallback")
    @Bulkhead(name = "payment-gateway")
    @TimeLimiter(name = "payment-gateway")
    public CompletableFuture<PaymentResult> charge(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> gateway.charge(request));
    }

    private CompletableFuture<PaymentResult> fallback(
            PaymentRequest request, Throwable t) {
        log.error("Payment gateway unavailable after all retries: {}", t.getMessage());
        return CompletableFuture.completedFuture(
            PaymentResult.deferred(request.idempotencyKey(),
                "Gateway unavailable — queued for retry"));
    }
}
```

### Programmatic Composition (Full Control)

```java
@Component
public class ResilientPaymentGateway {

    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    private final TimeLimiter timeLimiter;
    private final PaymentGateway gateway;

    public PaymentResult charge(PaymentRequest request) {
        // Compose decorators — outermost first
        Supplier<PaymentResult> decorated = Decorators
            .ofSupplier(() -> gateway.charge(request))
            .withTimeLimiter(timeLimiter, Executors.newVirtualThreadPerTaskExecutor())
            .withBulkhead(bulkhead)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withFallback(
                List.of(CallNotPermittedException.class),
                e -> PaymentResult.circuitOpen("Gateway circuit is open")
            )
            .withFallback(
                List.of(BulkheadFullException.class),
                e -> PaymentResult.overloaded("Too many concurrent requests")
            )
            .decorate();

        return Try.ofSupplier(decorated)
            .recover(TimeoutException.class,
                e -> PaymentResult.timeout("Gateway did not respond in time"))
            .recover(Exception.class,
                e -> PaymentResult.error("Unexpected error: " + e.getMessage()))
            .get();
    }
}
```

### application.yml — Complete Multi-Service Configuration

```yaml
resilience4j:
  retry:
    instances:
      payment-gateway:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        enable-randomized-wait: true
        randomized-wait-factor: 0.5
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.ValidationException
      fraud-service:
        max-attempts: 2
        wait-duration: 200ms

  circuitbreaker:
    instances:
      payment-gateway:
        register-health-indicator: true
        sliding-window-type: count-based
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        record-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
      fraud-service:
        sliding-window-size: 20
        failure-rate-threshold: 60
        wait-duration-in-open-state: 60s

  bulkhead:
    instances:
      payment-gateway:
        max-concurrent-calls: 75
        max-wait-duration: 500ms
      fraud-service:
        max-concurrent-calls: 50
        max-wait-duration: 300ms

  timelimiter:
    instances:
      payment-gateway:
        timeout-duration: 5s
        cancel-running-future: true
      fraud-service:
        timeout-duration: 2s
```

---

## 📨 Pattern 7: Dead-Letter & Poison Pill Handling

> **When:** A message/event cannot be processed after all retries. Don't lose it — park it for investigation.

### Kafka Dead-Letter Topology

```
Main Topic: payments.process
  │
  ├── Success → commit offset
  │
  └── Failure → Retry Topic: payments.process.retry-1 (1 min delay)
                  │
                  ├── Success → commit
                  │
                  └── Failure → Retry Topic: payments.process.retry-2 (5 min delay)
                                  │
                                  ├── Success → commit
                                  │
                                  └── Failure → DLT: payments.process.dlt
                                                  │
                                                  ├── Alert operations team
                                                  ├── Persist to MongoDB for investigation
                                                  └── Dashboard for manual replay
```

### Spring Kafka Retry with DLT

```java
@Configuration
public class KafkaRetryConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
        // Dead-letter publisher
        var dltHandler = new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(
                record.topic() + ".dlt", record.partition()));

        // Backoff: 1s, 5s, 15s, then DLT
        var backoff = new ExponentialBackOffWithMaxRetries(3);
        backoff.setInitialInterval(1000);
        backoff.setMultiplier(5.0);
        backoff.setMaxInterval(15000);

        var handler = new DefaultErrorHandler(dltHandler, backoff);

        // Non-retryable exceptions — send directly to DLT
        handler.addNotRetryableExceptions(
            ValidationException.class,
            JsonParseException.class,
            SerializationException.class
        );

        return handler;
    }
}
```

### Poison Pill Protection

```java
@Component
public class SafePaymentDeserializer implements Deserializer<PaymentEvent> {

    private final ObjectMapper mapper;

    @Override
    public PaymentEvent deserialize(String topic, byte[] data) {
        try {
            return mapper.readValue(data, PaymentEvent.class);
        } catch (Exception e) {
            // DON'T throw — it would block the partition forever
            log.error("Poison pill detected on topic={}, publishing raw bytes to DLT", topic, e);
            // Return a sentinel value that the consumer can route to DLT
            return new PaymentEvent.PoisonPill(data, e.getMessage());
        }
    }
}
```

---

## 📊 Pattern 8: Observability for Resilience

> **Rule:** Every resilience pattern MUST emit metrics. Invisible failures are production incidents.

### Key Metrics to Emit

```java
@Component
public class ResilienceMetrics {

    private final MeterRegistry registry;

    // Retry metrics
    public void recordRetry(String service, int attempt, boolean success) {
        registry.counter("resilience.retry",
            "service", service,
            "attempt", String.valueOf(attempt),
            "outcome", success ? "success" : "failure"
        ).increment();
    }

    // Circuit breaker state changes
    public void recordCircuitBreakerStateChange(String service, String fromState, String toState) {
        registry.counter("resilience.circuit_breaker.state_change",
            "service", service,
            "from", fromState,
            "to", toState
        ).increment();
    }

    // Fallback invocations
    public void recordFallback(String service, String reason) {
        registry.counter("resilience.fallback",
            "service", service,
            "reason", reason
        ).increment();
    }

    // Timeout breaches
    public void recordTimeout(String service, Duration actual, Duration limit) {
        registry.counter("resilience.timeout",
            "service", service
        ).increment();

        registry.timer("resilience.timeout.overage", "service", service)
            .record(actual.minus(limit));
    }
}
```

### Resilience4j Auto-Metrics (Spring Boot Actuator)

```yaml
# application.yml — enables automatic Micrometer metrics
management:
  metrics:
    distribution:
      percentiles-histogram:
        resilience4j.circuitbreaker.calls: true
        resilience4j.retry.calls: true
    tags:
      application: payment-service
  health:
    circuitbreakers:
      enabled: true      # /actuator/health shows circuit breaker states
    ratelimiters:
      enabled: true
```

### Alert Rules (Prometheus/Grafana)

```yaml
# Circuit breaker opened — immediate investigation
- alert: CircuitBreakerOpened
  expr: resilience4j_circuitbreaker_state{state="open"} == 1
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Circuit breaker {{ $labels.name }} is OPEN"

# High retry rate — degraded performance
- alert: HighRetryRate
  expr: rate(resilience4j_retry_calls_total{kind="retry"}[5m]) /
        rate(resilience4j_retry_calls_total[5m]) > 0.3
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: ">30% of calls to {{ $labels.name }} are retries"

# Fallback rate spiking
- alert: FallbackRateHigh
  expr: rate(resilience4j_circuitbreaker_calls_total{kind="not_permitted"}[5m]) > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Calls to {{ $labels.name }} are being rejected (circuit open)"

# DLT messages accumulating
- alert: DeadLetterQueueGrowing
  expr: kafka_consumer_group_lag{topic=~".*\\.dlt"} > 100
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Dead letter topic {{ $labels.topic }} has {{ $value }} unprocessed messages"
```

---

## 🌐 Pattern 9: Multi-Datacenter Resilience

> **Context:** Active-active or active-passive deployments with datacenter affinity.

### Datacenter Failover Strategy

```
Primary DC (US-EAST)                    Secondary DC (US-WEST)
┌─────────────────────┐                ┌─────────────────────┐
│  Payment Service    │                │  Payment Service    │
│  ┌───────────────┐  │                │  ┌───────────────┐  │
│  │ Circuit Breaker│  │   failover    │  │ Circuit Breaker│  │
│  │ → Local Gateway├──┼───────────────┼──► Remote Gateway│  │
│  └───────────────┘  │                │  └───────────────┘  │
│                     │                │                     │
│  Local MongoDB ◄────┼── replication──┼──► Local MongoDB    │
│  Local Kafka   ◄────┼── mirroring ──┼──► Local Kafka      │
│  Local Artemis ◄────┼── bridging  ──┼──► Local Artemis    │
└─────────────────────┘                └─────────────────────┘
```

### Datacenter-Aware Circuit Breaker

```java
@Component
public class DatacenterAwareGateway {

    private final CircuitBreaker primaryBreaker;
    private final CircuitBreaker secondaryBreaker;
    private final PaymentGateway primaryGateway;
    private final PaymentGateway secondaryGateway;

    public PaymentResult charge(PaymentRequest request) {
        // Try primary datacenter first
        try {
            return CircuitBreaker.decorateSupplier(primaryBreaker,
                () -> primaryGateway.charge(request)).get();
        } catch (CallNotPermittedException e) {
            log.warn("Primary DC circuit open, failing over to secondary");
        } catch (Exception e) {
            log.warn("Primary DC call failed: {}", e.getMessage());
        }

        // Failover to secondary
        return CircuitBreaker.decorateSupplier(secondaryBreaker,
            () -> secondaryGateway.charge(request)).get();
    }
}
```

---

## 🧩 Pattern 10: Saga & Compensating Transactions

> **When:** Distributed transactions spanning multiple services where traditional ACID isn't possible.

### Orchestration-Based Saga

```
PaymentSaga Orchestrator
  │
  ├── Step 1: Reserve funds (Account Service)
  │     └── Compensate: Release reservation
  │
  ├── Step 2: Fraud check (Fraud Service)
  │     └── Compensate: Clear fraud flag
  │
  ├── Step 3: Charge gateway (Payment Gateway)
  │     └── Compensate: Refund charge
  │
  └── Step 4: Send notification (Notification Service)
        └── Compensate: Send failure notification

On failure at Step 3:
  → Compensate Step 2 (clear fraud flag)
  → Compensate Step 1 (release reservation)
  → Record failure
  → Alert
```

### Saga Implementation Skeleton

```java
public sealed interface SagaStep<T> {
    record Execute<T>(String name, Supplier<T> action) implements SagaStep<T> {}
    record Compensate<T>(String name, Consumer<T> undo) implements SagaStep<T> {}
}

@Component
public class PaymentSagaOrchestrator {

    public PaymentResult execute(PaymentRequest request) {
        var completedSteps = new ArrayDeque<Runnable>(); // Compensation stack

        try {
            // Step 1
            var reservation = accountService.reserveFunds(request);
            completedSteps.push(() -> accountService.releaseReservation(reservation.id()));

            // Step 2
            var fraudResult = fraudService.check(request);
            completedSteps.push(() -> fraudService.clearFlag(request.id()));

            if (!fraudResult.approved()) {
                throw new FraudRejectedException(fraudResult.reason());
            }

            // Step 3
            var chargeResult = gatewayClient.charge(request);
            completedSteps.push(() -> gatewayClient.refund(chargeResult.transactionId()));

            // Step 4 — notification failure should NOT trigger compensation
            Try.run(() -> notificationService.sendSuccess(request))
               .onFailure(e -> log.warn("Notification failed, non-critical", e));

            return PaymentResult.success(chargeResult.transactionId());

        } catch (Exception e) {
            log.error("Saga failed at step, compensating. Error: {}", e.getMessage());
            compensate(completedSteps);
            return PaymentResult.failed(e.getMessage());
        }
    }

    private void compensate(Deque<Runnable> steps) {
        while (!steps.isEmpty()) {
            try {
                steps.pop().run();
            } catch (Exception e) {
                // Compensation failure — CRITICAL alert, needs manual intervention
                log.error("COMPENSATION FAILED — manual intervention required", e);
                alertService.critical("Saga compensation failure", e);
            }
        }
    }
}
```

---

## ⚡ Quick Reference: Which Pattern When?

| Symptom | Primary Pattern | Supporting Patterns |
|---------|----------------|---------------------|
| Intermittent connection failures | **Retry** | Timeout, Circuit Breaker |
| Downstream completely down | **Circuit Breaker** | Fallback, Retry (for recovery) |
| Slow downstream degrading your service | **Timeout** | Bulkhead, Circuit Breaker |
| One slow consumer starving others | **Bulkhead** | Timeout |
| Need degraded response over no response | **Fallback** | Circuit Breaker, Cache |
| Unprocessable messages blocking queue | **Dead Letter** | Poison Pill handler |
| Distributed transaction across services | **Saga** | Retry per step, Idempotency |
| Multi-DC deployment | **DC-aware Circuit Breaker** | Failover, Retry |
| Thundering herd after recovery | **Retry with jitter** | Bulkhead, Rate Limiter |

---

## 🚫 Fault Tolerance Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **No timeout** | Threads hang forever, pool exhaustion | ALWAYS set timeouts on every external call |
| **Retry without backoff** | Hammers failing service, prevents recovery | Exponential backoff + jitter |
| **Retry non-idempotent ops** | Duplicate payments, double charges | Add idempotency keys or don't retry |
| **Single circuit breaker for all services** | One bad service blocks all calls | Per-service (or per-operation) breakers |
| **Swallowing exceptions in fallback** | Silent data loss, invisible failures | Log + metrics + alert on every fallback |
| **Infinite retry** | Never gives up, resource leak | Max attempts + dead-letter |
| **Ignoring partial failures** | 3/5 succeeded, 2 failed — now what? | Track per-item outcomes, compensate as needed |
| **Retry on permanent errors (4xx)** | Wastes resources, delays real error handling | Classify errors before retry |
| **Circuit breaker with no metrics** | You won't know it's open until customers complain | Health indicator + alerting |
| **Timeout > caller's timeout** | Downstream timeout is useless if caller gives up first | Timeout budget cascades downstream |

---

## 💡 Golden Rules of Fault Tolerance

```
1. EVERY external call gets a timeout. No exceptions.
2. EVERY retry must be on an idempotent operation.
3. EVERY circuit breaker must emit metrics and alerts.
4. EVERY failed message must go to a dead-letter — never lose data.
5. EVERY fallback must be logged and measured.
6. Composition order matters: Retry → Circuit Breaker → Bulkhead → Timeout
7. Assume every network call will fail — design for it.
8. Fail FAST over fail SLOW — slow failures cascade.
9. Jitter prevents thundering herds — always add randomization to retries.
10. Test your resilience patterns — chaos engineering is not optional for zero-fault-tolerance systems.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Resilience4j 2.x / Kafka / MongoDB / Artemis*
