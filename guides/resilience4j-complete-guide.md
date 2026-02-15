# Effective Modern Resilience with Resilience4j

A comprehensive guide to building resilient systems using Resilience4j with Kafka, databases, REST APIs, and other integrations.

---

## Table of Contents

1. [Resilience Fundamentals](#resilience-fundamentals)
2. [Resilience4j Core Modules](#resilience4j-core-modules)
3. [Circuit Breaker Pattern](#circuit-breaker-pattern)
4. [Retry Pattern](#retry-pattern)
5. [Rate Limiter](#rate-limiter)
6. [Bulkhead Pattern](#bulkhead-pattern)
7. [Time Limiter](#time-limiter)
8. [Spring Boot Integration](#spring-boot-integration)
9. [REST API Resilience](#rest-api-resilience)
10. [Database Resilience](#database-resilience)
11. [Kafka Resilience](#kafka-resilience)
12. [Monitoring and Metrics](#monitoring-and-metrics)
13. [Best Practices](#best-practices)

---

## Resilience Fundamentals

### Why Resilience?

**Distributed systems face many failure modes:**

- Network timeouts and connection failures
- Service downtime and slow responses
- Resource exhaustion (threads, memory, connections)
- Cascading failures across services
- Partial system degradation

**Resilience4j provides patterns to handle these failures:**

1. **Circuit Breaker** - Prevent calling failing services
2. **Retry** - Automatically retry transient failures
3. **Rate Limiter** - Limit requests to prevent overload
4. **Bulkhead** - Isolate resources and prevent exhaustion
5. **Time Limiter** - Set timeouts for operations
6. **Cache** - Cache results to reduce load

### Maven Dependencies

```xml
<properties>
    <resilience4j.version>2.2.0</resilience4j.version>
</properties>

<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <!-- Individual Modules -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-circuitbreaker</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-retry</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-ratelimiter</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-bulkhead</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-timelimiter</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <!-- Metrics -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-micrometer</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    
    <!-- Spring AOP (required for annotations) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    
    <!-- Actuator for monitoring -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

---

## Circuit Breaker Pattern

### Concept

The Circuit Breaker prevents cascading failures by stopping calls to a failing service.

**States:**
- **CLOSED**: Normal operation, calls pass through
- **OPEN**: Too many failures, calls are blocked
- **HALF_OPEN**: Testing if service recovered

### Basic Implementation

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;

public class CircuitBreakerExample {
    
    public CircuitBreaker createCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                    // Open at 50% failure rate
            .slowCallRateThreshold(50)                   // Open at 50% slow calls
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(60))  // Wait before half-open
            .permittedNumberOfCallsInHalfOpenState(10)   // Test with 10 calls
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(100)                      // Consider last 100 calls
            .minimumNumberOfCalls(10)                    // Min calls before calculating
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .recordExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();
        
        return CircuitBreaker.of("externalService", config);
    }
    
    public String callWithCircuitBreaker(CircuitBreaker cb) {
        return CircuitBreaker.decorateSupplier(cb, () -> {
            return externalService.call();
        }).get();
    }
    
    public String callWithFallback(CircuitBreaker cb) {
        try {
            return CircuitBreaker.decorateSupplier(cb, 
                () -> externalService.call()
            ).get();
        } catch (Exception e) {
            logger.error("Circuit breaker fallback", e);
            return "Fallback response";
        }
    }
}
```

### Event Listeners

```java
public void setupEventListeners(CircuitBreaker cb) {
    cb.getEventPublisher()
        .onStateTransition(event -> {
            logger.warn("Circuit {} changed: {} -> {}",
                event.getCircuitBreakerName(),
                event.getStateTransition().getFromState(),
                event.getStateTransition().getToState());
            
            if (event.getStateTransition().getToState() == State.OPEN) {
                alertService.sendAlert("Circuit breaker opened!");
            }
        })
        .onSuccess(event -> logger.debug("Call succeeded"))
        .onError(event -> logger.warn("Call failed", event.getThrowable()))
        .onCallNotPermitted(event -> 
            logger.warn("Call rejected - circuit open"));
}
```

---

## Retry Pattern

### Basic Retry

```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;

public class RetryExample {
    
    public Retry createRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();
        
        return Retry.of("externalService", config);
    }
    
    public String callWithRetry(Retry retry) {
        return Retry.decorateSupplier(retry, () -> {
            logger.info("Attempting call");
            return externalService.call();
        }).get();
    }
}
```

### Exponential Backoff

```java
public Retry createRetryWithBackoff() {
    RetryConfig config = RetryConfig.custom()
        .maxAttempts(5)
        .intervalFunction(IntervalFunction.ofExponentialBackoff(
            Duration.ofMillis(100),  // Initial delay
            2.0                       // Multiplier: 100ms, 200ms, 400ms, 800ms
        ))
        .retryOnException(e -> 
            e instanceof IOException ||
            (e instanceof HttpException && 
             ((HttpException) e).getStatusCode() >= 500)
        )
        .build();
    
    return Retry.of("withBackoff", config);
}
```

### Random Jitter

```java
public Retry createRetryWithJitter() {
    RetryConfig config = RetryConfig.custom()
        .maxAttempts(5)
        .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
            Duration.ofMillis(100),
            2.0,
            0.5  // Randomization factor (prevents thundering herd)
        ))
        .build();
    
    return Retry.of("withJitter", config);
}
```

### Event Listeners

```java
public void setupRetryListeners(Retry retry) {
    retry.getEventPublisher()
        .onRetry(event -> {
            logger.warn("Retry attempt {} after {}ms",
                event.getNumberOfRetryAttempts(),
                event.getWaitInterval().toMillis());
        })
        .onSuccess(event -> {
            logger.info("Success after {} attempts",
                event.getNumberOfRetryAttempts());
        })
        .onError(event -> {
            logger.error("All retries failed",
                event.getLastThrowable());
        });
}
```

---

## Rate Limiter

### Basic Rate Limiter

```java
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;

public class RateLimiterExample {
    
    public RateLimiter createRateLimiter() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(10)                      // 10 calls
            .limitRefreshPeriod(Duration.ofSeconds(1))  // per second
            .timeoutDuration(Duration.ofMillis(500))    // Wait max 500ms
            .build();
        
        return RateLimiter.of("externalService", config);
    }
    
    public String callWithRateLimit(RateLimiter rl) {
        return RateLimiter.decorateSupplier(rl, () -> {
            return externalService.call();
        }).get();
    }
}
```

### Per-User Rate Limiting

```java
public class PerUserRateLimiting {
    
    private final Map<String, RateLimiter> userLimiters = new ConcurrentHashMap<>();
    
    public RateLimiter getRateLimiterForUser(String userId) {
        return userLimiters.computeIfAbsent(userId, id -> {
            RateLimiterConfig config = RateLimiterConfig.custom()
                .limitForPeriod(100)
                .limitRefreshPeriod(Duration.ofMinutes(1))
                .build();
            
            return RateLimiter.of("user-" + id, config);
        });
    }
    
    public String callWithUserLimit(String userId) {
        RateLimiter rl = getRateLimiterForUser(userId);
        return RateLimiter.decorateSupplier(rl, 
            () -> processRequest(userId)
        ).get();
    }
}
```

---

## Bulkhead Pattern

### Semaphore Bulkhead

```java
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;

public class BulkheadExample {
    
    public Bulkhead createBulkhead() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)                  // Max 10 concurrent
            .maxWaitDuration(Duration.ofMillis(500)) // Wait max 500ms
            .build();
        
        return Bulkhead.of("externalService", config);
    }
    
    public String callWithBulkhead(Bulkhead bulkhead) {
        return Bulkhead.decorateSupplier(bulkhead, () -> {
            return externalService.call();
        }).get();
    }
}
```

### Thread Pool Bulkhead

```java
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;

public class ThreadPoolBulkheadExample {
    
    public ThreadPoolBulkhead createThreadPoolBulkhead() {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(20)
            .keepAliveDuration(Duration.ofMillis(1000))
            .build();
        
        return ThreadPoolBulkhead.of("externalService", config);
    }
    
    public CompletableFuture<String> callAsync(ThreadPoolBulkhead bulkhead) {
        return bulkhead.executeSupplier(() -> externalService.call());
    }
}
```

---

## Time Limiter

```java
import io.github.resilience4j.timelimiter.TimeLimiter;
import io.github.resilience4j.timelimiter.TimeLimiterConfig;

public class TimeLimiterExample {
    
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public TimeLimiter createTimeLimiter() {
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2))
            .cancelRunningFuture(true)
            .build();
        
        return TimeLimiter.of("externalService", config);
    }
    
    public String callWithTimeout(TimeLimiter tl) throws Exception {
        Supplier<CompletableFuture<String>> futureSupplier = () ->
            CompletableFuture.supplyAsync(() -> {
                return externalService.call();
            }, executor);
        
        Callable<String> callable = TimeLimiter.decorateFutureSupplier(
            tl, futureSupplier
        );
        
        return callable.call();
    }
}
```

---

---

## Programmatic Usage (Without Spring)

### Standalone Setup

```java
// Maven dependencies (no Spring required)
<dependencies>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-all</artifactId>
        <version>2.2.0</version>
    </dependency>
</dependencies>
```

---

## Circuit Breaker - Programmatic

### Basic Usage

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

public class CircuitBreakerStandalone {
    
    private final ExternalServiceClient client;
    private final CircuitBreaker circuitBreaker;
    
    public CircuitBreakerStandalone(ExternalServiceClient client) {
        this.client = client;
        
        // Create configuration
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .slowCallRateThreshold(50)
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .permittedNumberOfCallsInHalfOpenState(10)
            .slidingWindowSize(100)
            .minimumNumberOfCalls(10)
            .build();
        
        // Create circuit breaker
        this.circuitBreaker = CircuitBreaker.of("externalService", config);
        
        // Setup event listeners
        setupEventListeners();
    }
    
    // Decorate supplier
    public String callService() {
        Supplier<String> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> client.getData());
        
        return decoratedSupplier.get();
    }
    
    // Decorate function
    public String callServiceWithParam(String param) {
        Function<String, String> decoratedFunction = CircuitBreaker
            .decorateFunction(circuitBreaker, p -> client.getData(p));
        
        return decoratedFunction.apply(param);
    }
    
    // Decorate callable
    public String callServiceChecked() throws Exception {
        Callable<String> decoratedCallable = CircuitBreaker
            .decorateCallable(circuitBreaker, () -> client.getDataChecked());
        
        return decoratedCallable.call();
    }
    
    // Decorate runnable
    public void performAction() {
        Runnable decoratedRunnable = CircuitBreaker
            .decorateRunnable(circuitBreaker, () -> client.performAction());
        
        decoratedRunnable.run();
    }
    
    // With fallback using Try
    public String callWithFallback() {
        return Try.ofSupplier(
            CircuitBreaker.decorateSupplier(circuitBreaker, () -> client.getData())
        )
        .recover(throwable -> {
            logger.error("Circuit breaker fallback", throwable);
            return "Fallback response";
        })
        .get();
    }
    
    // Manual control
    public void manualControl() {
        // Get current state
        CircuitBreaker.State state = circuitBreaker.getState();
        
        // Manually transition states
        circuitBreaker.transitionToOpenState();
        circuitBreaker.transitionToClosedState();
        circuitBreaker.transitionToHalfOpenState();
        
        // Reset circuit breaker
        circuitBreaker.reset();
        
        // Get metrics
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        float failureRate = metrics.getFailureRate();
        int failedCalls = metrics.getNumberOfFailedCalls();
        int successfulCalls = metrics.getNumberOfSuccessfulCalls();
    }
    
    private void setupEventListeners() {
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                logger.info("State transition: {} -> {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()))
            .onSuccess(event -> 
                logger.debug("Call succeeded"))
            .onError(event -> 
                logger.error("Call failed", event.getThrowable()))
            .onCallNotPermitted(event -> 
                logger.warn("Call not permitted - circuit open"));
    }
}
```

### Using Registry Pattern

```java
public class CircuitBreakerRegistryExample {
    
    private final CircuitBreakerRegistry registry;
    
    public CircuitBreakerRegistryExample() {
        // Create default registry
        this.registry = CircuitBreakerRegistry.ofDefaults();
        
        // Or with custom default config
        CircuitBreakerConfig defaultConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .build();
        
        this.registry = CircuitBreakerRegistry.of(defaultConfig);
    }
    
    public String callService1() {
        CircuitBreaker cb = registry.circuitBreaker("service1");
        
        return CircuitBreaker.decorateSupplier(cb, () -> 
            externalService1.call()
        ).get();
    }
    
    public String callService2() {
        // Get or create with custom config
        CircuitBreakerConfig customConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(30)
            .build();
        
        CircuitBreaker cb = registry.circuitBreaker("service2", customConfig);
        
        return CircuitBreaker.decorateSupplier(cb, () -> 
            externalService2.call()
        ).get();
    }
    
    // Access all circuit breakers
    public void monitorAllCircuitBreakers() {
        registry.getAllCircuitBreakers().forEach(cb -> {
            logger.info("Circuit breaker: {}, State: {}, Metrics: {}",
                cb.getName(),
                cb.getState(),
                cb.getMetrics());
        });
    }
}
```

---

## Retry - Programmatic

### Basic Usage

```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;

public class RetryStandalone {
    
    private final ExternalServiceClient client;
    private final Retry retry;
    
    public RetryStandalone(ExternalServiceClient client) {
        this.client = client;
        
        // Create configuration
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();
        
        // Create retry
        this.retry = Retry.of("externalService", config);
        
        // Setup event listeners
        setupEventListeners();
    }
    
    // Decorate supplier
    public String callService() {
        Supplier<String> decoratedSupplier = Retry
            .decorateSupplier(retry, () -> client.getData());
        
        return decoratedSupplier.get();
    }
    
    // Decorate function
    public String callServiceWithParam(String param) {
        Function<String, String> decoratedFunction = Retry
            .decorateFunction(retry, p -> client.getData(p));
        
        return decoratedFunction.apply(param);
    }
    
    // Decorate callable
    public String callServiceChecked() throws Exception {
        Callable<String> decoratedCallable = Retry
            .decorateCallable(retry, () -> client.getDataChecked());
        
        return decoratedCallable.call();
    }
    
    // With CompletionStage
    public CompletionStage<String> callServiceAsync() {
        Supplier<CompletionStage<String>> supplier = 
            () -> client.getDataAsync();
        
        return Retry.decorateCompletionStage(
            retry,
            Executors.newSingleThreadScheduledExecutor(),
            supplier
        ).get();
    }
    
    private void setupEventListeners() {
        retry.getEventPublisher()
            .onRetry(event -> 
                logger.info("Retry attempt {} after {}ms",
                    event.getNumberOfRetryAttempts(),
                    event.getWaitInterval().toMillis()))
            .onSuccess(event -> 
                logger.info("Success after {} attempts",
                    event.getNumberOfRetryAttempts()))
            .onError(event -> 
                logger.error("All retries failed", event.getLastThrowable()));
    }
}
```

### Exponential Backoff

```java
public class RetryWithBackoff {
    
    private final Retry retry;
    
    public RetryWithBackoff() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(100),  // Initial interval
                2.0                       // Multiplier
            ))
            .build();
        
        this.retry = Retry.of("withBackoff", config);
    }
    
    public String callWithBackoff() {
        return Retry.decorateSupplier(retry, () -> 
            externalService.call()
        ).get();
    }
}
```

### Random Jitter

```java
public class RetryWithJitter {
    
    private final Retry retry;
    
    public RetryWithJitter() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(100),
                2.0,
                0.5  // Randomization factor
            ))
            .build();
        
        this.retry = Retry.of("withJitter", config);
    }
    
    public String callWithJitter() {
        return Retry.decorateSupplier(retry, () -> 
            externalService.call()
        ).get();
    }
}
```

---

## Rate Limiter - Programmatic

### Basic Usage

```java
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import io.github.resilience4j.ratelimiter.RateLimiterRegistry;

public class RateLimiterStandalone {
    
    private final ExternalServiceClient client;
    private final RateLimiter rateLimiter;
    
    public RateLimiterStandalone(ExternalServiceClient client) {
        this.client = client;
        
        // Create configuration
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(10)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ofMillis(500))
            .build();
        
        // Create rate limiter
        this.rateLimiter = RateLimiter.of("externalService", config);
    }
    
    // Decorate supplier
    public String callService() {
        Supplier<String> decoratedSupplier = RateLimiter
            .decorateSupplier(rateLimiter, () -> client.getData());
        
        return decoratedSupplier.get();
    }
    
    // Check permission manually
    public String callWithPermissionCheck() {
        boolean permitted = rateLimiter.acquirePermission();
        
        if (permitted) {
            try {
                return client.getData();
            } finally {
                // Permission consumed automatically
            }
        } else {
            throw new RequestNotPermittedException(
                "Rate limit exceeded", rateLimiter);
        }
    }
    
    // Try to acquire permission with timeout
    public String callWithTimeout(Duration timeout) {
        boolean permitted = rateLimiter.acquirePermission(timeout);
        
        if (permitted) {
            return client.getData();
        } else {
            throw new RequestNotPermittedException(
                "Rate limit exceeded after timeout", rateLimiter);
        }
    }
    
    // Get metrics
    public void printMetrics() {
        RateLimiter.Metrics metrics = rateLimiter.getMetrics();
        
        int available = metrics.getAvailablePermissions();
        int waiting = metrics.getNumberOfWaitingThreads();
        
        logger.info("Available: {}, Waiting: {}", available, waiting);
    }
}
```

### Per-User Rate Limiting

```java
public class PerUserRateLimiter {
    
    private final Map<String, RateLimiter> userLimiters = new ConcurrentHashMap<>();
    private final RateLimiterConfig config;
    
    public PerUserRateLimiter() {
        this.config = RateLimiterConfig.custom()
            .limitForPeriod(100)
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .build();
    }
    
    public String callForUser(String userId) {
        RateLimiter limiter = userLimiters.computeIfAbsent(
            userId,
            id -> RateLimiter.of("user-" + id, config)
        );
        
        return RateLimiter.decorateSupplier(limiter, () -> 
            processRequest(userId)
        ).get();
    }
    
    // Cleanup inactive users
    public void cleanupInactiveUsers() {
        userLimiters.entrySet().removeIf(entry -> {
            RateLimiter limiter = entry.getValue();
            return limiter.getMetrics().getAvailablePermissions() == 
                   config.getLimitForPeriod();
        });
    }
}
```

---

## Bulkhead - Programmatic

### Semaphore Bulkhead

```java
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;

public class BulkheadStandalone {
    
    private final ExternalServiceClient client;
    private final Bulkhead bulkhead;
    
    public BulkheadStandalone(ExternalServiceClient client) {
        this.client = client;
        
        // Create configuration
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
        
        // Create bulkhead
        this.bulkhead = Bulkhead.of("externalService", config);
    }
    
    // Decorate supplier
    public String callService() {
        Supplier<String> decoratedSupplier = Bulkhead
            .decorateSupplier(bulkhead, () -> client.getData());
        
        return decoratedSupplier.get();
    }
    
    // Try to execute with permission check
    public String callWithPermissionCheck() {
        if (bulkhead.tryAcquirePermission()) {
            try {
                return client.getData();
            } finally {
                bulkhead.releasePermission();
            }
        } else {
            throw new BulkheadFullException(
                "Bulkhead is full", bulkhead);
        }
    }
    
    // Get metrics
    public void printMetrics() {
        Bulkhead.Metrics metrics = bulkhead.getMetrics();
        
        int available = metrics.getAvailableConcurrentCalls();
        int max = metrics.getMaxAllowedConcurrentCalls();
        
        logger.info("Bulkhead: {}/{} available", available, max);
    }
}
```

### Thread Pool Bulkhead

```java
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;

public class ThreadPoolBulkheadStandalone {
    
    private final ExternalServiceClient client;
    private final ThreadPoolBulkhead bulkhead;
    
    public ThreadPoolBulkheadStandalone(ExternalServiceClient client) {
        this.client = client;
        
        // Create configuration
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(20)
            .keepAliveDuration(Duration.ofMillis(1000))
            .build();
        
        // Create thread pool bulkhead
        this.bulkhead = ThreadPoolBulkhead.of("externalService", config);
    }
    
    // Execute async
    public CompletableFuture<String> callServiceAsync() {
        Supplier<String> supplier = () -> client.getData();
        
        return bulkhead.executeSupplier(supplier);
    }
    
    // Execute with timeout
    public String callServiceWithTimeout(Duration timeout) 
            throws Exception {
        
        CompletableFuture<String> future = bulkhead.executeSupplier(
            () -> client.getData()
        );
        
        return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
    }
    
    // Get metrics
    public void printMetrics() {
        ThreadPoolBulkhead.Metrics metrics = bulkhead.getMetrics();
        
        int corePoolSize = metrics.getCoreThreadPoolSize();
        int currentPoolSize = metrics.getThreadPoolSize();
        int queueDepth = metrics.getQueueDepth();
        int queueCapacity = metrics.getQueueCapacity();
        
        logger.info("Pool: {}/{}, Queue: {}/{}",
            currentPoolSize, corePoolSize,
            queueDepth, queueCapacity);
    }
}
```

---

## Time Limiter - Programmatic

```java
import io.github.resilience4j.timelimiter.TimeLimiter;
import io.github.resilience4j.timelimiter.TimeLimiterConfig;

public class TimeLimiterStandalone {
    
    private final ExternalServiceClient client;
    private final TimeLimiter timeLimiter;
    private final ExecutorService executor;
    
    public TimeLimiterStandalone(ExternalServiceClient client) {
        this.client = client;
        this.executor = Executors.newCachedThreadPool();
        
        // Create configuration
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2))
            .cancelRunningFuture(true)
            .build();
        
        // Create time limiter
        this.timeLimiter = TimeLimiter.of("externalService", config);
    }
    
    // Decorate CompletableFuture supplier
    public String callService() throws Exception {
        Supplier<CompletableFuture<String>> futureSupplier = () ->
            CompletableFuture.supplyAsync(() -> client.getData(), executor);
        
        Callable<String> callable = TimeLimiter.decorateFutureSupplier(
            timeLimiter,
            futureSupplier
        );
        
        return callable.call();
    }
    
    // With fallback on timeout
    public String callWithFallback() {
        try {
            return callService();
        } catch (TimeoutException e) {
            logger.warn("Call timed out", e);
            return "Fallback response";
        } catch (Exception e) {
            logger.error("Call failed", e);
            throw new RuntimeException(e);
        }
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

---

## Combining Multiple Patterns - Programmatic

### Using Decorators

```java
import io.github.resilience4j.decorators.Decorators;

public class CombinedPatterns {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final RateLimiter rateLimiter;
    private final Bulkhead bulkhead;
    
    public CombinedPatterns() {
        // Create all patterns
        this.circuitBreaker = createCircuitBreaker();
        this.retry = createRetry();
        this.rateLimiter = createRateLimiter();
        this.bulkhead = createBulkhead();
    }
    
    // Combine: Circuit Breaker + Retry + Rate Limiter
    public String callWithMultiplePatterns() {
        Supplier<String> supplier = () -> externalService.call();
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withFallback(Arrays.asList(
                IOException.class,
                TimeoutException.class,
                CallNotPermittedException.class
            ), e -> {
                logger.error("All patterns failed", e);
                return "Fallback response";
            })
            .get();
    }
    
    // Combine: All patterns
    public String callWithAllPatterns() {
        Supplier<String> supplier = () -> externalService.call();
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .withFallback(throwable -> "Fallback response")
            .get();
    }
    
    // Combine: For checked exceptions
    public String callChecked() throws Exception {
        Callable<String> callable = () -> externalService.callChecked();
        
        return Decorators.ofCallable(callable)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .decorate()
            .call();
    }
    
    // Combine: Async with CompletionStage
    public CompletionStage<String> callAsync() {
        Supplier<CompletionStage<String>> supplier = 
            () -> externalService.callAsync();
        
        ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(1);
        
        return Decorators.ofCompletionStage(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry, scheduler)
            .withRateLimiter(rateLimiter)
            .withFallback(Arrays.asList(
                IOException.class,
                TimeoutException.class
            ), e -> CompletableFuture.completedFuture("Fallback"))
            .get();
    }
    
    private CircuitBreaker createCircuitBreaker() {
        return CircuitBreaker.of("service", CircuitBreakerConfig.ofDefaults());
    }
    
    private Retry createRetry() {
        return Retry.of("service", RetryConfig.ofDefaults());
    }
    
    private RateLimiter createRateLimiter() {
        return RateLimiter.of("service", RateLimiterConfig.ofDefaults());
    }
    
    private Bulkhead createBulkhead() {
        return Bulkhead.of("service", BulkheadConfig.ofDefaults());
    }
}
```

---

## Real-World Standalone Examples

### HTTP Client with Full Resilience

```java
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class ResilientHttpClient {
    
    private final HttpClient httpClient;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final RateLimiter rateLimiter;
    private final TimeLimiter timeLimiter;
    private final ExecutorService executor;
    
    public ResilientHttpClient(String baseUrl) {
        // Create HTTP client
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
        
        this.executor = Executors.newCachedThreadPool();
        
        // Create resilience patterns
        this.circuitBreaker = createCircuitBreaker();
        this.retry = createRetry();
        this.rateLimiter = createRateLimiter();
        this.timeLimiter = createTimeLimiter();
    }
    
    public String get(String url) {
        Supplier<String> supplier = () -> {
            try {
                HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .GET()
                    .build();
                
                HttpResponse<String> response = httpClient.send(
                    request,
                    HttpResponse.BodyHandlers.ofString()
                );
                
                if (response.statusCode() >= 500) {
                    throw new IOException("Server error: " + response.statusCode());
                }
                
                return response.body();
            } catch (IOException | InterruptedException e) {
                throw new RuntimeException(e);
            }
        };
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withFallback(throwable -> {
                logger.error("HTTP GET failed", throwable);
                return getCachedResponse(url);
            })
            .get();
    }
    
    public CompletableFuture<String> getAsync(String url) {
        Supplier<CompletableFuture<String>> futureSupplier = () -> {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET()
                .build();
            
            return httpClient.sendAsync(
                request,
                HttpResponse.BodyHandlers.ofString()
            ).thenApply(HttpResponse::body);
        };
        
        try {
            Callable<CompletableFuture<String>> callable = 
                TimeLimiter.decorateFutureSupplier(timeLimiter, futureSupplier);
            
            return Decorators.ofCallable(callable)
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .withRateLimiter(rateLimiter)
                .decorate()
                .call();
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
    
    private CircuitBreaker createCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .slidingWindowSize(100)
            .recordExceptions(IOException.class, TimeoutException.class)
            .build();
        
        return CircuitBreaker.of("httpClient", config);
    }
    
    private Retry createRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(100), 2.0
            ))
            .retryExceptions(IOException.class, TimeoutException.class)
            .build();
        
        return Retry.of("httpClient", config);
    }
    
    private RateLimiter createRateLimiter() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(100)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .build();
        
        return RateLimiter.of("httpClient", config);
    }
    
    private TimeLimiter createTimeLimiter() {
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(5))
            .cancelRunningFuture(true)
            .build();
        
        return TimeLimiter.of("httpClient", config);
    }
    
    private String getCachedResponse(String url) {
        // Return cached response or default
        return "Cached response";
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

### Database Connection Pool with Resilience

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.Statement;

public class ResilientDatabaseClient {
    
    private final HikariDataSource dataSource;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    
    public ResilientDatabaseClient(String jdbcUrl, String username, String password) {
        // Create connection pool
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        
        this.dataSource = new HikariDataSource(config);
        
        // Create resilience patterns
        this.circuitBreaker = createCircuitBreaker();
        this.retry = createRetry();
        this.bulkhead = createBulkhead();
    }
    
    public List<User> findAllUsers() {
        Supplier<List<User>> supplier = () -> {
            try (Connection conn = dataSource.getConnection();
                 Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
                
                List<User> users = new ArrayList<>();
                while (rs.next()) {
                    users.add(new User(
                        rs.getString("id"),
                        rs.getString("name"),
                        rs.getString("email")
                    ));
                }
                return users;
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        };
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withBulkhead(bulkhead)
            .withFallback(Arrays.asList(SQLException.class), e -> {
                logger.error("Database query failed", e);
                return Collections.emptyList();
            })
            .get();
    }
    
    public void saveUser(User user) {
        Runnable runnable = () -> {
            try (Connection conn = dataSource.getConnection();
                 PreparedStatement stmt = conn.prepareStatement(
                     "INSERT INTO users (id, name, email) VALUES (?, ?, ?)"
                 )) {
                
                stmt.setString(1, user.getId());
                stmt.setString(2, user.getName());
                stmt.setString(3, user.getEmail());
                stmt.executeUpdate();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        };
        
        Decorators.ofRunnable(runnable)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withBulkhead(bulkhead)
            .decorate()
            .run();
    }
    
    private CircuitBreaker createCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(50)
            .recordExceptions(SQLException.class)
            .build();
        
        return CircuitBreaker.of("database", config);
    }
    
    private Retry createRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(2)
            .waitDuration(Duration.ofMillis(100))
            .retryExceptions(SQLException.class)
            .build();
        
        return Retry.of("database", config);
    }
    
    private Bulkhead createBulkhead() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(20)
            .build();
        
        return Bulkhead.of("database", config);
    }
    
    public void shutdown() {
        dataSource.close();
    }
}
```

### Message Queue Consumer with Resilience

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

public class ResilientKafkaConsumer {
    
    private final KafkaConsumer<String, String> consumer;
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    private final ExecutorService executor;
    
    public ResilientKafkaConsumer(String bootstrapServers, String groupId) {
        // Create Kafka consumer
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        
        this.consumer = new KafkaConsumer<>(props);
        this.executor = Executors.newFixedThreadPool(10);
        
        // Create resilience patterns
        this.circuitBreaker = createCircuitBreaker();
        this.retry = createRetry();
        this.bulkhead = createBulkhead();
    }
    
    public void subscribe(String topic) {
        consumer.subscribe(Collections.singletonList(topic));
        
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, String> record : records) {
                processRecordWithResilience(record);
            }
            
            consumer.commitSync();
        }
    }
    
    private void processRecordWithResilience(ConsumerRecord<String, String> record) {
        Runnable runnable = () -> processRecord(record);
        
        Decorators.ofRunnable(runnable)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withBulkhead(bulkhead)
            .withFallback(throwable -> {
                logger.error("Failed to process record, sending to DLQ", throwable);
                sendToDLQ(record);
            })
            .decorate()
            .run();
    }
    
    private void processRecord(ConsumerRecord<String, String> record) {
        logger.info("Processing: {}", record.value());
        // Process message
        businessLogic.process(record.value());
    }
    
    private void sendToDLQ(ConsumerRecord<String, String> record) {
        // Send to dead letter queue
        dlqProducer.send(record.value());
    }
    
    private CircuitBreaker createCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .build();
        
        return CircuitBreaker.of("kafkaConsumer", config);
    }
    
    private Retry createRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(1))
            .build();
        
        return Retry.of("kafkaConsumer", config);
    }
    
    private Bulkhead createBulkhead() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .build();
        
        return Bulkhead.of("kafkaConsumer", config);
    }
    
    public void shutdown() {
        consumer.close();
        executor.shutdown();
    }
}
```

### Service Orchestration with Multiple Dependencies

```java
public class OrderService {
    
    private final ResilientHttpClient inventoryClient;
    private final ResilientHttpClient paymentClient;
    private final ResilientHttpClient notificationClient;
    private final ResilientDatabaseClient database;
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    
    public OrderService() {
        this.inventoryClient = new ResilientHttpClient("http://inventory-service");
        this.paymentClient = new ResilientHttpClient("http://payment-service");
        this.notificationClient = new ResilientHttpClient("http://notification-service");
        this.database = new ResilientDatabaseClient("jdbc:postgresql://localhost/orders", "user", "pass");
        
        // Overall orchestration resilience
        this.circuitBreaker = createCircuitBreaker();
        this.retry = createRetry();
    }
    
    public Order createOrder(OrderRequest request) {
        Supplier<Order> supplier = () -> {
            // Step 1: Check inventory
            String inventoryResponse = inventoryClient.get(
                "/inventory/check?productId=" + request.getProductId() +
                "&quantity=" + request.getQuantity()
            );
            
            if (!inventoryResponse.contains("available")) {
                throw new InsufficientInventoryException();
            }
            
            // Step 2: Process payment
            String paymentResponse = paymentClient.get(
                "/payment/process?amount=" + request.getAmount()
            );
            
            if (!paymentResponse.contains("success")) {
                throw new PaymentFailedException();
            }
            
            // Step 3: Create order in database
            Order order = new Order(request);
            database.saveOrder(order);
            
            // Step 4: Send notification (fire and forget)
            CompletableFuture.runAsync(() -> {
                try {
                    notificationClient.get(
                        "/notification/send?orderId=" + order.getId()
                    );
                } catch (Exception e) {
                    logger.error("Notification failed", e);
                }
            });
            
            return order;
        };
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withFallback(throwable -> {
                logger.error("Order creation failed", throwable);
                
                // Compensating actions
                try {
                    inventoryClient.get("/inventory/release?productId=" + 
                        request.getProductId());
                } catch (Exception e) {
                    logger.error("Failed to release inventory", e);
                }
                
                throw new OrderCreationException(
                    "Unable to create order. Please try again.", throwable);
            })
            .get();
    }
    
    private CircuitBreaker createCircuitBreaker() {
        return CircuitBreaker.of("orderService", CircuitBreakerConfig.ofDefaults());
    }
    
    private Retry createRetry() {
        return Retry.of("orderService", RetryConfig.ofDefaults());
    }
    
    public void shutdown() {
        inventoryClient.shutdown();
        paymentClient.shutdown();
        notificationClient.shutdown();
        database.shutdown();
    }
}
```

---

## Standalone Configuration Builder

```java
public class ResilienceConfigBuilder {
    
    public static class CircuitBreakerBuilder {
        private final CircuitBreakerConfig.Builder builder;
        
        public CircuitBreakerBuilder() {
            this.builder = CircuitBreakerConfig.custom();
        }
        
        public CircuitBreakerBuilder failureRate(float threshold) {
            builder.failureRateThreshold(threshold);
            return this;
        }
        
        public CircuitBreakerBuilder slowCallRate(float threshold, Duration duration) {
            builder.slowCallRateThreshold(threshold);
            builder.slowCallDurationThreshold(duration);
            return this;
        }
        
        public CircuitBreakerBuilder waitInOpen(Duration duration) {
            builder.waitDurationInOpenState(duration);
            return this;
        }
        
        public CircuitBreakerBuilder slidingWindow(int size) {
            builder.slidingWindowSize(size);
            return this;
        }
        
        public CircuitBreakerBuilder recordExceptions(Class<? extends Throwable>... exceptions) {
            builder.recordExceptions(exceptions);
            return this;
        }
        
        public CircuitBreaker build(String name) {
            return CircuitBreaker.of(name, builder.build());
        }
    }
    
    public static class RetryBuilder {
        private final RetryConfig.Builder builder;
        
        public RetryBuilder() {
            this.builder = RetryConfig.custom();
        }
        
        public RetryBuilder maxAttempts(int attempts) {
            builder.maxAttempts(attempts);
            return this;
        }
        
        public RetryBuilder exponentialBackoff(Duration initial, double multiplier) {
            builder.intervalFunction(
                IntervalFunction.ofExponentialBackoff(initial, multiplier)
            );
            return this;
        }
        
        public RetryBuilder retryExceptions(Class<? extends Throwable>... exceptions) {
            builder.retryExceptions(exceptions);
            return this;
        }
        
        public Retry build(String name) {
            return Retry.of(name, builder.build());
        }
    }
    
    // Usage example
    public static void main(String[] args) {
        CircuitBreaker cb = new CircuitBreakerBuilder()
            .failureRate(50)
            .slowCallRate(50, Duration.ofSeconds(2))
            .waitInOpen(Duration.ofSeconds(60))
            .slidingWindow(100)
            .recordExceptions(IOException.class, TimeoutException.class)
            .build("myService");
        
        Retry retry = new RetryBuilder()
            .maxAttempts(3)
            .exponentialBackoff(Duration.ofMillis(100), 2.0)
            .retryExceptions(IOException.class)
            .build("myService");
    }
}
```

## Spring Boot Integration

### Configuration (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      externalService:
        register-health-indicator: true
        sliding-window-size: 100
        sliding-window-type: COUNT_BASED
        minimum-number-of-calls: 10
        permitted-number-of-calls-in-half-open-state: 10
        wait-duration-in-open-state: 60s
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BusinessException
  
  retry:
    instances:
      externalService:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
  
  ratelimiter:
    instances:
      externalService:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 500ms
  
  bulkhead:
    instances:
      externalService:
        max-concurrent-calls: 10
        max-wait-duration: 500ms
  
  timelimiter:
    instances:
      externalService:
        timeout-duration: 2s
        cancel-running-future: true

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers
  health:
    circuitbreakers:
      enabled: true
```

### Annotation-Based Usage

```java
@Service
public class ExternalServiceClient {
    
    @CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
    @Retry(name = "externalService")
    public String call() {
        logger.info("Calling external service");
        return restTemplate.getForObject(
            "http://external-service/api/data", 
            String.class
        );
    }
    
    private String fallback(Exception e) {
        logger.error("Fallback triggered", e);
        return "Cached response";
    }
    
    @RateLimiter(name = "externalService")
    public String callWithRateLimit() {
        return restTemplate.getForObject(
            "http://external-service/api/data", 
            String.class
        );
    }
    
    @Bulkhead(name = "externalService", type = Bulkhead.Type.SEMAPHORE)
    public String callWithBulkhead() {
        return restTemplate.getForObject(
            "http://external-service/api/data", 
            String.class
        );
    }
    
    // Combining multiple patterns
    @CircuitBreaker(name = "externalService", fallbackMethod = "complexFallback")
    @Retry(name = "externalService")
    @RateLimiter(name = "externalService")
    @Bulkhead(name = "externalService")
    public String complexCall() {
        return restTemplate.getForObject(
            "http://external-service/api/data", 
            String.class
        );
    }
    
    private String complexFallback(Exception e) {
        logger.error("All patterns failed", e);
        return "Emergency fallback";
    }
}
```

### Programmatic Usage

```java
@Service
public class ProgrammaticResilience {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final RateLimiter rateLimiter;
    
    public ProgrammaticResilience(
            CircuitBreakerRegistry cbRegistry,
            RetryRegistry retryRegistry,
            RateLimiterRegistry rlRegistry) {
        
        this.circuitBreaker = cbRegistry.circuitBreaker("externalService");
        this.retry = retryRegistry.retry("externalService");
        this.rateLimiter = rlRegistry.rateLimiter("externalService");
    }
    
    public String call() {
        Supplier<String> supplier = () -> externalService.call();
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withFallback(Arrays.asList(
                IOException.class, 
                TimeoutException.class
            ), e -> "Fallback response")
            .get();
    }
}
```

---

## REST API Resilience

### RestTemplate with Resilience

```java
@Service
public class ResilientRestClient {
    
    private final RestTemplate restTemplate;
    
    @CircuitBreaker(name = "externalApi", fallbackMethod = "getFallback")
    @Retry(name = "externalApi")
    public <T> T get(String url, Class<T> responseType) {
        return restTemplate.getForObject(url, responseType);
    }
    
    private <T> T getFallback(String url, Class<T> type, Exception e) {
        logger.error("GET failed for " + url, e);
        return null;
    }
    
    @CircuitBreaker(name = "externalApi")
    @Retry(name = "externalApi")
    public <T, R> R post(String url, T request, Class<R> responseType) {
        return restTemplate.postForObject(url, request, responseType);
    }
}
```

### WebClient (Reactive) with Resilience

```java
@Service
public class ResilientWebClient {
    
    private final WebClient webClient;
    
    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallbackGet")
    @Retry(name = "externalApi")
    public Mono<String> get(String path) {
        return webClient.get()
            .uri(path)
            .retrieve()
            .bodyToMono(String.class)
            .timeout(Duration.ofSeconds(2));
    }
    
    private Mono<String> fallbackGet(String path, Exception e) {
        logger.error("GET failed for " + path, e);
        return Mono.just("Fallback response");
    }
}
```

### Feign Client with Resilience

```java
@FeignClient(
    name = "external-service",
    url = "${external.service.url}"
)
public interface ExternalServiceClient {
    @GetMapping("/api/data")
    String getData();
}

@Service
public class ResilientFeignService {
    
    private final ExternalServiceClient feignClient;
    
    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallbackGetData")
    @Retry(name = "externalApi")
    @Bulkhead(name = "externalApi")
    public String getData() {
        return feignClient.getData();
    }
    
    private String fallbackGetData(Exception e) {
        logger.error("Feign call failed", e);
        return "Fallback data";
    }
}
```

---

## Database Resilience

### JDBC with Resilience

```java
@Service
public class ResilientDatabaseService {
    
    private final JdbcTemplate jdbcTemplate;
    
    @CircuitBreaker(name = "database", fallbackMethod = "fallbackQuery")
    @Retry(name = "database")
    public List<User> findUsers() {
        return jdbcTemplate.query(
            "SELECT * FROM users",
            (rs, rowNum) -> new User(
                rs.getString("id"),
                rs.getString("name")
            )
        );
    }
    
    private List<User> fallbackQuery(Exception e) {
        logger.error("Database query failed", e);
        return Collections.emptyList();
    }
    
    @Retry(name = "database")
    @Bulkhead(name = "database")
    public void saveUser(User user) {
        jdbcTemplate.update(
            "INSERT INTO users (id, name) VALUES (?, ?)",
            user.getId(), user.getName()
        );
    }
}
```

### JPA with Resilience

```java
@Service
public class ResilientJpaService {
    
    private final UserRepository repository;
    
    @CircuitBreaker(name = "database", fallbackMethod = "fallbackFind")
    @Retry(name = "database")
    public Optional<User> findById(String id) {
        return repository.findById(id);
    }
    
    private Optional<User> fallbackFind(String id, Exception e) {
        logger.error("Failed to find user: " + id, e);
        return Optional.empty();
    }
    
    @CircuitBreaker(name = "database")
    @Retry(name = "database")
    public User save(User user) {
        return repository.save(user);
    }
}
```

### Connection Pool Configuration

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000  # 30 seconds
      idle-timeout: 600000       # 10 minutes
      max-lifetime: 1800000      # 30 minutes
      connection-test-query: SELECT 1
      validation-timeout: 5000
```

---

## Kafka Resilience

### Producer with Resilience

```java
@Service
public class ResilientKafkaProducer {
    
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @CircuitBreaker(name = "kafka", fallbackMethod = "fallbackSend")
    @Retry(name = "kafka")
    @Bulkhead(name = "kafka")
    public void sendMessage(String topic, String message) {
        kafkaTemplate.send(topic, message)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    logger.error("Failed to send to Kafka", ex);
                } else {
                    logger.info("Sent to partition: {}",
                        result.getRecordMetadata().partition());
                }
            });
    }
    
    private void fallbackSend(String topic, String message, Exception e) {
        logger.error("Kafka send failed, storing to DLQ", e);
        deadLetterQueueService.store(topic, message);
    }
}
```

### Consumer with Resilience

```java
@Service
public class ResilientKafkaConsumer {
    
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    @CircuitBreaker(name = "kafka", fallbackMethod = "fallbackConsume")
    @Retry(name = "kafka")
    public void consume(ConsumerRecord<String, String> record) {
        logger.info("Consuming: {}", record.value());
        processMessage(record.value());
    }
    
    private void fallbackConsume(ConsumerRecord<String, String> record, Exception e) {
        logger.error("Failed to process message", e);
        kafkaTemplate.send("my-topic.DLQ", record.value());
    }
}
```

### Kafka Configuration for Resilience

```yaml
spring:
  kafka:
    producer:
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
    
    consumer:
      enable-auto-commit: false
      max-poll-interval-ms: 300000
      session-timeout-ms: 10000
```

---

## Monitoring and Metrics

### Metrics Configuration

```java
@Configuration
public class ResilienceMetricsConfig {
    
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
    
    @Bean
    public CircuitBreakerMetrics circuitBreakerMetrics(
            CircuitBreakerRegistry registry,
            MeterRegistry meterRegistry) {
        
        registry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher().onStateTransition(event -> {
                meterRegistry.counter(
                    "resilience4j.circuitbreaker.state.transition",
                    "name", event.getCircuitBreakerName(),
                    "from", event.getStateTransition().getFromState().name(),
                    "to", event.getStateTransition().getToState().name()
                ).increment();
            });
        });
        
        return new CircuitBreakerMetrics(registry);
    }
}
```

### Health Indicators

```java
@Component
public class ResilienceHealthIndicator implements HealthIndicator {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Override
    public Health health() {
        Map<String, State> states = new HashMap<>();
        
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb -> {
            states.put(cb.getName(), cb.getState());
        });
        
        boolean hasOpen = states.values().stream()
            .anyMatch(state -> state == State.OPEN);
        
        if (hasOpen) {
            return Health.down()
                .withDetail("circuitBreakers", states)
                .build();
        }
        
        return Health.up()
            .withDetail("circuitBreakers", states)
            .build();
    }
}
```

### Event Logging

```java
@Component
public class ResilienceEventLogger {
    
    @PostConstruct
    public void registerEventListeners() {
        // Circuit breaker events
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    logger.warn("Circuit {} transitioned: {} -> {}",
                        event.getCircuitBreakerName(),
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState());
                    
                    if (event.getStateTransition().getToState() == State.OPEN) {
                        alertService.sendAlert(
                            "Circuit Breaker Opened",
                            "Circuit " + event.getCircuitBreakerName() + " is OPEN"
                        );
                    }
                })
                .onFailureRateExceeded(event -> {
                    logger.error("Failure rate {}% exceeded for {}",
                        event.getFailureRate(),
                        event.getCircuitBreakerName());
                });
        });
        
        // Retry events
        retryRegistry.getAllRetries().forEach(retry -> {
            retry.getEventPublisher()
                .onRetry(event -> {
                    logger.info("Retry {} attempt {}",
                        event.getName(),
                        event.getNumberOfRetryAttempts());
                })
                .onError(event -> {
                    logger.error("All retries failed for {}",
                        event.getName(),
                        event.getLastThrowable());
                });
        });
    }
}
```

---

## Best Practices

### ✅ DO

1. **Start with circuit breaker** - Prevent cascading failures first
2. **Add retry for transient failures** - Network issues, temporary problems
3. **Use exponential backoff** - Prevent thundering herd
4. **Set appropriate timeouts** - Don't wait forever
5. **Implement meaningful fallbacks** - Cache, default values, degraded service
6. **Monitor metrics** - Track success/failure rates, circuit states
7. **Test failure scenarios** - Chaos engineering, fault injection
8. **Use bulkhead for isolation** - Protect critical resources
9. **Configure per service** - Different services need different settings
10. **Combine patterns wisely** - Circuit breaker + retry + fallback

### ❌ DON'T

1. **Don't retry indefinitely** - Always set max attempts
2. **Don't use same config everywhere** - Customize per service
3. **Don't ignore slow calls** - They're as bad as failures
4. **Don't retry non-transient errors** - Business exceptions, 4xx errors
5. **Don't forget fallbacks** - Always have plan B
6. **Don't over-complicate** - Start simple, add complexity as needed
7. **Don't ignore metrics** - Monitor and adjust thresholds
8. **Don't retry without backoff** - Use exponential backoff
9. **Don't test only happy path** - Test failure modes
10. **Don't leave circuits always open** - Monitor and investigate

### Pattern Combinations

**Circuit Breaker + Retry + Fallback:**
```java
@CircuitBreaker(name = "service", fallbackMethod = "fallback")
@Retry(name = "service")
public String call() {
    return externalService.call();
}

private String fallback(Exception e) {
    return "Cached response";
}
```

**Rate Limiter + Bulkhead:**
```java
@RateLimiter(name = "api")
@Bulkhead(name = "api")
public Response handleRequest(Request request) {
    return process(request);
}
```

**All Patterns:**
```java
@CircuitBreaker(name = "comprehensive", fallbackMethod = "fallback")
@Retry(name = "comprehensive")
@RateLimiter(name = "comprehensive")
@Bulkhead(name = "comprehensive")
@TimeLimiter(name = "comprehensive")
public CompletableFuture<String> callWithAllPatterns() {
    return CompletableFuture.supplyAsync(() -> service.call());
}
```

---

## Real-World Example: E-Commerce Order Service

```java
@Service
public class OrderProcessingService {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @Transactional
    @CircuitBreaker(name = "orderProcessing", fallbackMethod = "fallbackOrder")
    @Retry(name = "orderProcessing")
    @Bulkhead(name = "orderProcessing")
    public Order processOrder(OrderRequest request) {
        logger.info("Processing order: {}", request.getOrderId());
        
        // Step 1: Reserve inventory
        reserveInventory(request);
        
        // Step 2: Process payment
        PaymentResult payment = processPayment(request);
        
        // Step 3: Create order
        Order order = createOrder(request, payment);
        
        // Step 4: Send notification (async)
        sendConfirmation(order);
        
        return order;
    }
    
    @CircuitBreaker(name = "inventory", fallbackMethod = "fallbackInventory")
    @Retry(name = "inventory")
    private void reserveInventory(OrderRequest request) {
        inventoryService.reserve(
            request.getProductId(),
            request.getQuantity()
        );
    }
    
    private void fallbackInventory(OrderRequest request, Exception e) {
        logger.error("Inventory reservation failed", e);
        throw new InventoryException("Unable to reserve inventory", e);
    }
    
    @CircuitBreaker(name = "payment", fallbackMethod = "fallbackPayment")
    @Retry(name = "payment")
    @RateLimiter(name = "payment")
    private PaymentResult processPayment(OrderRequest request) {
        return paymentService.processPayment(
            request.getCustomerId(),
            request.getAmount()
        );
    }
    
    private PaymentResult fallbackPayment(OrderRequest request, Exception e) {
        logger.error("Payment failed", e);
        throw new PaymentException("Unable to process payment", e);
    }
    
    @CircuitBreaker(name = "notification")
    @Retry(name = "notification")
    private void sendConfirmation(Order order) {
        try {
            notificationService.sendConfirmation(order);
        } catch (Exception e) {
            logger.warn("Notification failed, queuing for retry", e);
            notificationQueue.add(order);
        }
    }
    
    private Order fallbackOrder(OrderRequest request, Exception e) {
        logger.error("Order processing failed completely", e);
        
        // Compensating transaction
        try {
            inventoryService.release(
                request.getProductId(),
                request.getQuantity()
            );
        } catch (Exception ex) {
            logger.error("Failed to release inventory", ex);
        }
        
        throw new OrderException(
            "Unable to process order. Please try again later.",
            e
        );
    }
}
```

### Configuration for Order Service

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderProcessing:
        sliding-window-size: 100
        failure-rate-threshold: 50
        wait-duration-in-open-state: 60s
      
      inventory:
        sliding-window-size: 50
        failure-rate-threshold: 60
        wait-duration-in-open-state: 30s
      
      payment:
        sliding-window-size: 100
        failure-rate-threshold: 40
        wait-duration-in-open-state: 120s
      
      notification:
        sliding-window-size: 20
        failure-rate-threshold: 70
        wait-duration-in-open-state: 30s
  
  retry:
    instances:
      orderProcessing:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
      
      inventory:
        max-attempts: 2
        wait-duration: 500ms
      
      payment:
        max-attempts: 3
        wait-duration: 2s
      
      notification:
        max-attempts: 5
        wait-duration: 1s
  
  bulkhead:
    instances:
      orderProcessing:
        max-concurrent-calls: 50
  
  ratelimiter:
    instances:
      payment:
        limit-for-period: 100
        limit-refresh-period: 1m
```

---

## Conclusion

**Key Resilience Patterns:**

1. **Circuit Breaker** - Stop calling failing services
2. **Retry** - Handle transient failures
3. **Rate Limiter** - Prevent overload
4. **Bulkhead** - Isolate resources
5. **Time Limiter** - Set timeouts
6. **Fallback** - Graceful degradation

**When to Use Each Pattern:**

- **External service calls** → Circuit Breaker + Retry + Fallback
- **Transient network issues** → Retry with exponential backoff
- **Service overload** → Rate Limiter + Bulkhead
- **Slow responses** → Time Limiter + Circuit Breaker
- **Resource exhaustion** → Bulkhead
- **Critical operations** → Combine all patterns

**Remember:**
- Resilience is about graceful degradation
- Monitor metrics and adjust thresholds
- Test failure scenarios
- Start simple, add complexity as needed
- Provide meaningful fallbacks

Resilience4j makes it easy to build robust, production-ready systems that handle failures gracefully!

