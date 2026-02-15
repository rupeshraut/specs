# Resilience4j - Programmatic Usage (Without Spring)

Complete guide for using Resilience4j programmatically without Spring framework dependencies.

---

## Table of Contents

1. [Standalone Setup](#standalone-setup)
2. [Circuit Breaker - Programmatic](#circuit-breaker-programmatic)
3. [Retry - Programmatic](#retry-programmatic)
4. [Rate Limiter - Programmatic](#rate-limiter-programmatic)
5. [Bulkhead - Programmatic](#bulkhead-programmatic)
6. [Time Limiter - Programmatic](#time-limiter-programmatic)
7. [Combining Patterns](#combining-patterns-programmatic)
8. [Custom Registries](#custom-registries)
9. [Event Publishing](#event-publishing-programmatic)
10. [Complete Examples](#complete-standalone-examples)

---

## Standalone Setup

### Maven Dependencies (No Spring)

```xml
<dependencies>
    <!-- Core Resilience4j modules (NO Spring dependencies) -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-circuitbreaker</artifactId>
        <version>2.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-retry</artifactId>
        <version>2.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-ratelimiter</artifactId>
        <version>2.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-bulkhead</artifactId>
        <version>2.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-timelimiter</artifactId>
        <version>2.2.0</version>
    </dependency>
    
    <!-- Optional: For decorators -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-all</artifactId>
        <version>2.2.0</version>
    </dependency>
</dependencies>
```

---

## Circuit Breaker - Programmatic

### Basic Circuit Breaker

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

public class CircuitBreakerExample {
    
    public static void main(String[] args) {
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
        CircuitBreaker circuitBreaker = CircuitBreaker.of("externalService", config);
        
        // Decorate your call
        Supplier<String> decoratedSupplier = CircuitBreaker.decorateSupplier(
            circuitBreaker,
            () -> callExternalService()
        );
        
        // Execute
        try {
            String result = decoratedSupplier.get();
            System.out.println("Result: " + result);
        } catch (Exception e) {
            System.err.println("Circuit breaker rejected call: " + e.getMessage());
        }
    }
    
    private static String callExternalService() {
        // Your external service call
        return "Success";
    }
}
```

### Circuit Breaker with Runnable

```java
public class CircuitBreakerRunnableExample {
    
    public void executeWithCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.ofDefaults();
        CircuitBreaker circuitBreaker = CircuitBreaker.of("service", config);
        
        // Decorate Runnable
        Runnable decoratedRunnable = CircuitBreaker.decorateRunnable(
            circuitBreaker,
            () -> {
                System.out.println("Executing task");
                performTask();
            }
        );
        
        // Execute
        try {
            decoratedRunnable.run();
        } catch (Exception e) {
            System.err.println("Execution failed: " + e.getMessage());
        }
    }
    
    private void performTask() {
        // Task implementation
    }
}
```

### Circuit Breaker with Callable

```java
public class CircuitBreakerCallableExample {
    
    public String executeWithCircuitBreaker() throws Exception {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .build();
        
        CircuitBreaker circuitBreaker = CircuitBreaker.of("service", config);
        
        // Decorate Callable
        Callable<String> decoratedCallable = CircuitBreaker.decorateCallable(
            circuitBreaker,
            () -> {
                return fetchData();
            }
        );
        
        // Execute
        return decoratedCallable.call();
    }
    
    private String fetchData() throws Exception {
        // Fetch data logic
        return "Data";
    }
}
```

### Circuit Breaker with CompletionStage

```java
public class CircuitBreakerAsyncExample {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public CompletableFuture<String> executeAsync() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("asyncService");
        
        // Decorate CompletionStage
        Supplier<CompletionStage<String>> stageSupplier = () ->
            CompletableFuture.supplyAsync(() -> callService(), executor);
        
        Supplier<CompletionStage<String>> decoratedSupplier =
            CircuitBreaker.decorateCompletionStage(circuitBreaker, stageSupplier);
        
        return decoratedSupplier.get().toCompletableFuture();
    }
    
    private String callService() {
        return "Async result";
    }
}
```

### Circuit Breaker with Try-Catch Recovery

```java
public class CircuitBreakerWithRecovery {
    
    public String executeWithFallback() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
        
        try {
            return CircuitBreaker.decorateSupplier(
                circuitBreaker,
                this::callExternalService
            ).get();
        } catch (CallNotPermittedException e) {
            // Circuit is open
            return handleCircuitOpen(e);
        } catch (Exception e) {
            // Other failures
            return handleFailure(e);
        }
    }
    
    private String callExternalService() {
        // External call
        return "Success";
    }
    
    private String handleCircuitOpen(CallNotPermittedException e) {
        System.err.println("Circuit is OPEN, using fallback");
        return "Cached response";
    }
    
    private String handleFailure(Exception e) {
        System.err.println("Call failed: " + e.getMessage());
        return "Default response";
    }
}
```

---

## Retry - Programmatic

### Basic Retry

```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;

public class RetryExample {
    
    public String executeWithRetry() {
        // Create retry configuration
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();
        
        // Create retry
        Retry retry = Retry.of("service", config);
        
        // Decorate supplier
        Supplier<String> decoratedSupplier = Retry.decorateSupplier(
            retry,
            () -> callService()
        );
        
        // Execute
        return decoratedSupplier.get();
    }
    
    private String callService() throws IOException {
        // Service call that may fail
        if (Math.random() > 0.5) {
            throw new IOException("Temporary failure");
        }
        return "Success";
    }
}
```

### Retry with Exponential Backoff

```java
public class RetryWithBackoffExample {
    
    public String executeWithBackoff() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(100),  // Initial interval
                2.0                       // Multiplier
            ))
            .build();
        
        Retry retry = Retry.of("backoffService", config);
        
        Supplier<String> retryableSupplier = Retry.decorateSupplier(
            retry,
            this::unstableService
        );
        
        return retryableSupplier.get();
    }
    
    private String unstableService() {
        System.out.println("Attempting service call at " + System.currentTimeMillis());
        // May fail, will retry with increasing delays: 100ms, 200ms, 400ms, 800ms
        return "Result";
    }
}
```

### Retry with Random Jitter

```java
public class RetryWithJitterExample {
    
    public String executeWithJitter() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(100),
                2.0,
                0.5  // Randomization factor (prevents thundering herd)
            ))
            .build();
        
        Retry retry = Retry.of("jitterService", config);
        
        return Retry.decorateSupplier(retry, this::callService).get();
    }
    
    private String callService() {
        return "Result";
    }
}
```

### Retry with Custom Predicate

```java
public class RetryWithPredicateExample {
    
    public Response executeWithCustomRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .retryOnResult(response -> {
                // Retry if response indicates retry
                if (response instanceof HttpResponse) {
                    int status = ((HttpResponse) response).getStatusCode();
                    return status == 503 || status == 429;
                }
                return false;
            })
            .retryOnException(e -> {
                // Retry only on specific exceptions
                return e instanceof SocketTimeoutException ||
                       e instanceof ConnectException;
            })
            .build();
        
        Retry retry = Retry.of("customService", config);
        
        return Retry.decorateSupplier(retry, this::makeHttpCall).get();
    }
    
    private Response makeHttpCall() {
        return new HttpResponse(200, "OK");
    }
}
```

### Retry with Runnable

```java
public class RetryRunnableExample {
    
    public void executeTaskWithRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(1))
            .build();
        
        Retry retry = Retry.of("task", config);
        
        Runnable decoratedRunnable = Retry.decorateRunnable(
            retry,
            () -> {
                System.out.println("Executing task");
                performTask();
            }
        );
        
        decoratedRunnable.run();
    }
    
    private void performTask() {
        // Task that may fail
    }
}
```

---

## Rate Limiter - Programmatic

### Basic Rate Limiter

```java
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;

public class RateLimiterExample {
    
    public String executeWithRateLimit() {
        // Create configuration: 10 calls per second
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(10)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ofMillis(500))
            .build();
        
        RateLimiter rateLimiter = RateLimiter.of("service", config);
        
        // Decorate supplier
        Supplier<String> decoratedSupplier = RateLimiter.decorateSupplier(
            rateLimiter,
            () -> callService()
        );
        
        // Execute
        try {
            return decoratedSupplier.get();
        } catch (RequestNotPermitted e) {
            return "Rate limit exceeded, try again later";
        }
    }
    
    private String callService() {
        return "Service result";
    }
}
```

### Rate Limiter with Manual Permission

```java
public class RateLimiterManualExample {
    
    public void executeWithManualCheck() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(100)
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .build();
        
        RateLimiter rateLimiter = RateLimiter.of("api", config);
        
        // Check if call is permitted
        if (rateLimiter.acquirePermission()) {
            try {
                String result = callApi();
                System.out.println("Result: " + result);
            } catch (Exception e) {
                System.err.println("API call failed: " + e.getMessage());
            }
        } else {
            System.err.println("Rate limit exceeded");
        }
    }
    
    private String callApi() {
        return "API response";
    }
}
```

### Rate Limiter with Timeout

```java
public class RateLimiterWithTimeoutExample {
    
    public String executeWithTimeout() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(5)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ofMillis(100))  // Wait max 100ms
            .build();
        
        RateLimiter rateLimiter = RateLimiter.of("service", config);
        
        // Try to acquire permission with timeout
        boolean permitted = rateLimiter.acquirePermission(1);
        
        if (permitted) {
            return callService();
        } else {
            throw new RateLimitExceededException("Service overloaded");
        }
    }
    
    private String callService() {
        return "Result";
    }
}
```

---

## Bulkhead - Programmatic

### Semaphore Bulkhead

```java
import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;

public class BulkheadExample {
    
    public String executeWithBulkhead() {
        // Create configuration: max 10 concurrent calls
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
        
        Bulkhead bulkhead = Bulkhead.of("service", config);
        
        // Decorate supplier
        Supplier<String> decoratedSupplier = Bulkhead.decorateSupplier(
            bulkhead,
            () -> callService()
        );
        
        // Execute
        try {
            return decoratedSupplier.get();
        } catch (BulkheadFullException e) {
            return "Service busy, try again later";
        }
    }
    
    private String callService() {
        // Simulate slow operation
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "Result";
    }
}
```

### Thread Pool Bulkhead

```java
import io.github.resilience4j.bulkhead.ThreadPoolBulkhead;
import io.github.resilience4j.bulkhead.ThreadPoolBulkheadConfig;

public class ThreadPoolBulkheadExample {
    
    public CompletableFuture<String> executeAsync() {
        // Create thread pool bulkhead
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(20)
            .keepAliveDuration(Duration.ofMillis(1000))
            .build();
        
        ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("service", config);
        
        // Execute asynchronously
        Supplier<String> supplier = () -> callService();
        
        return bulkhead.executeSupplier(supplier);
    }
    
    private String callService() {
        return "Async result";
    }
}
```

---

## Time Limiter - Programmatic

### Basic Time Limiter

```java
import io.github.resilience4j.timelimiter.TimeLimiter;
import io.github.resilience4j.timelimiter.TimeLimiterConfig;

public class TimeLimiterExample {
    
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public String executeWithTimeout() throws Exception {
        // Create time limiter: 2 second timeout
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2))
            .cancelRunningFuture(true)
            .build();
        
        TimeLimiter timeLimiter = TimeLimiter.of("service", config);
        
        // Create future supplier
        Supplier<CompletableFuture<String>> futureSupplier = () ->
            CompletableFuture.supplyAsync(() -> callService(), executor);
        
        // Decorate and execute
        Callable<String> callable = TimeLimiter.decorateFutureSupplier(
            timeLimiter,
            futureSupplier
        );
        
        return callable.call();
    }
    
    private String callService() {
        // Potentially slow operation
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "Result";
    }
}
```

---

## Combining Patterns Programmatic

### Circuit Breaker + Retry

```java
public class CombinedCircuitBreakerRetry {
    
    public String executeWithCombinedPatterns() {
        // Create circuit breaker
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
        
        // Create retry
        Retry retry = Retry.ofDefaults("service");
        
        // Combine: Circuit breaker wraps retry
        Supplier<String> decoratedSupplier = Decorators.ofSupplier(() -> callService())
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .decorate();
        
        // Execute
        try {
            return decoratedSupplier.get();
        } catch (Exception e) {
            System.err.println("All patterns failed: " + e.getMessage());
            return "Fallback response";
        }
    }
    
    private String callService() {
        return "Success";
    }
}
```

### All Patterns Combined

```java
import io.github.resilience4j.decorators.Decorators;

public class AllPatternsCombined {
    
    public String executeWithAllPatterns() {
        // Create all patterns
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
        Retry retry = Retry.ofDefaults("service");
        RateLimiter rateLimiter = RateLimiter.ofDefaults("service");
        Bulkhead bulkhead = Bulkhead.ofDefaults("service");
        
        // Combine all patterns
        Supplier<String> decoratedSupplier = Decorators.ofSupplier(() -> callService())
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .withFallback(
                Arrays.asList(Exception.class),
                e -> "Fallback: " + e.getMessage()
            )
            .decorate();
        
        return decoratedSupplier.get();
    }
    
    private String callService() {
        return "Success";
    }
}
```

### CompletableFuture with Multiple Patterns

```java
public class AsyncCombinedPatterns {
    
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(10);
    
    public CompletableFuture<String> executeAsyncWithPatterns() {
        // Create patterns
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
        Retry retry = Retry.ofDefaults("service");
        TimeLimiter timeLimiter = TimeLimiter.of(
            "service",
            TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(3))
                .build()
        );
        
        // Create supplier
        Supplier<CompletableFuture<String>> supplier = () ->
            CompletableFuture.supplyAsync(() -> callService(), scheduler);
        
        // Combine patterns
        return Decorators.ofCompletionStage(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry, scheduler)
            .withTimeLimiter(timeLimiter, scheduler)
            .get()
            .toCompletableFuture();
    }
    
    private String callService() {
        return "Async result";
    }
}
```

---

## Custom Registries

### Creating Custom Registry

```java
public class CustomRegistryExample {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;
    private final RateLimiterRegistry rateLimiterRegistry;
    
    public CustomRegistryExample() {
        // Create custom circuit breaker registry
        CircuitBreakerConfig cbConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .build();
        
        this.circuitBreakerRegistry = CircuitBreakerRegistry.of(cbConfig);
        
        // Create custom retry registry
        RetryConfig retryConfig = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .build();
        
        this.retryRegistry = RetryRegistry.of(retryConfig);
        
        // Create custom rate limiter registry
        RateLimiterConfig rlConfig = RateLimiterConfig.custom()
            .limitForPeriod(10)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .build();
        
        this.rateLimiterRegistry = RateLimiterRegistry.of(rlConfig);
    }
    
    public String executeWithCustomRegistry(String serviceName) {
        // Get or create circuit breaker from registry
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker(serviceName);
        
        // Get or create retry from registry
        Retry retry = retryRegistry.retry(serviceName);
        
        // Get or create rate limiter from registry
        RateLimiter rl = rateLimiterRegistry.rateLimiter(serviceName);
        
        // Use them
        return Decorators.ofSupplier(() -> callService(serviceName))
            .withCircuitBreaker(cb)
            .withRetry(retry)
            .withRateLimiter(rl)
            .decorate()
            .get();
    }
    
    private String callService(String serviceName) {
        return "Result from " + serviceName;
    }
}
```

### Registry with Per-Service Configuration

```java
public class PerServiceRegistryExample {
    
    private final CircuitBreakerRegistry registry;
    
    public PerServiceRegistryExample() {
        // Default configuration
        CircuitBreakerConfig defaultConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .build();
        
        // Create registry with default config
        registry = CircuitBreakerRegistry.of(defaultConfig);
        
        // Add custom configuration for specific service
        CircuitBreakerConfig criticalServiceConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(30)  // More sensitive
            .waitDurationInOpenState(Duration.ofSeconds(120))
            .build();
        
        registry.addConfiguration("criticalService", criticalServiceConfig);
    }
    
    public String callNormalService() {
        CircuitBreaker cb = registry.circuitBreaker("normalService");
        return CircuitBreaker.decorateSupplier(cb, () -> "Normal result").get();
    }
    
    public String callCriticalService() {
        CircuitBreaker cb = registry.circuitBreaker(
            "criticalService",
            "criticalService"  // Uses custom config
        );
        return CircuitBreaker.decorateSupplier(cb, () -> "Critical result").get();
    }
}
```

---

## Event Publishing Programmatic

### Circuit Breaker Events

```java
public class CircuitBreakerEventExample {
    
    public void setupCircuitBreakerEvents() {
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
        
        // Listen to state transitions
        circuitBreaker.getEventPublisher().onStateTransition(event -> {
            System.out.println("State changed: " + 
                event.getStateTransition().getFromState() + " -> " +
                event.getStateTransition().getToState());
        });
        
        // Listen to success events
        circuitBreaker.getEventPublisher().onSuccess(event -> {
            System.out.println("Call succeeded");
        });
        
        // Listen to error events
        circuitBreaker.getEventPublisher().onError(event -> {
            System.err.println("Call failed: " + event.getThrowable().getMessage());
        });
        
        // Listen to call not permitted events
        circuitBreaker.getEventPublisher().onCallNotPermitted(event -> {
            System.err.println("Call blocked - circuit is open");
        });
        
        // Listen to slow call events
        circuitBreaker.getEventPublisher().onSlowCallRateExceeded(event -> {
            System.err.println("Slow call rate exceeded: " + event.getSlowCallRate());
        });
        
        // Listen to failure rate events
        circuitBreaker.getEventPublisher().onFailureRateExceeded(event -> {
            System.err.println("Failure rate exceeded: " + event.getFailureRate());
        });
    }
}
```

### Retry Events

```java
public class RetryEventExample {
    
    public void setupRetryEvents() {
        Retry retry = Retry.ofDefaults("service");
        
        // Listen to retry events
        retry.getEventPublisher().onRetry(event -> {
            System.out.println("Retry attempt: " + 
                event.getNumberOfRetryAttempts() +
                " after " + event.getWaitInterval().toMillis() + "ms");
        });
        
        // Listen to success after retry
        retry.getEventPublisher().onSuccess(event -> {
            System.out.println("Succeeded after " + 
                event.getNumberOfRetryAttempts() + " attempts");
        });
        
        // Listen to error after all retries
        retry.getEventPublisher().onError(event -> {
            System.err.println("All " + event.getNumberOfRetryAttempts() + 
                " retry attempts failed");
        });
        
        // Listen to ignored errors
        retry.getEventPublisher().onIgnoredError(event -> {
            System.out.println("Error ignored (not retryable): " + 
                event.getLastThrowable().getClass().getSimpleName());
        });
    }
}
```

### Custom Event Consumer

```java
public class CustomEventConsumer {
    
    public void setupCustomEventHandling() {
        CircuitBreaker cb = CircuitBreaker.ofDefaults("service");
        
        // Custom event consumer
        cb.getEventPublisher().onEvent(event -> {
            switch (event.getEventType()) {
                case SUCCESS:
                    handleSuccess(event);
                    break;
                case ERROR:
                    handleError(event);
                    break;
                case STATE_TRANSITION:
                    handleStateTransition(event);
                    break;
                case NOT_PERMITTED:
                    handleNotPermitted(event);
                    break;
            }
        });
    }
    
    private void handleSuccess(CircuitBreakerEvent event) {
        System.out.println("Success: " + event.getElapsedDuration().toMillis() + "ms");
    }
    
    private void handleError(CircuitBreakerEvent event) {
        System.err.println("Error: " + event.getThrowable().getMessage());
    }
    
    private void handleStateTransition(CircuitBreakerEvent event) {
        CircuitBreakerOnStateTransitionEvent stateEvent = 
            (CircuitBreakerOnStateTransitionEvent) event;
        System.out.println("State: " + stateEvent.getStateTransition());
    }
    
    private void handleNotPermitted(CircuitBreakerEvent event) {
        System.err.println("Call not permitted - circuit open");
    }
}
```

---

## Complete Standalone Examples

### REST API Client with Full Resilience

```java
public class ResilientHttpClient {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final RateLimiter rateLimiter;
    private final Bulkhead bulkhead;
    private final HttpClient httpClient;
    
    public ResilientHttpClient() {
        // Initialize HTTP client
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
        
        // Circuit breaker: open after 50% failures
        CircuitBreakerConfig cbConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .slowCallRateThreshold(50)
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .permittedNumberOfCallsInHalfOpenState(10)
            .slidingWindowSize(100)
            .build();
        this.circuitBreaker = CircuitBreaker.of("api", cbConfig);
        
        // Retry: 3 attempts with exponential backoff
        RetryConfig retryConfig = RetryConfig.custom()
            .maxAttempts(3)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(100),
                2.0
            ))
            .retryExceptions(IOException.class, TimeoutException.class)
            .build();
        this.retry = Retry.of("api", retryConfig);
        
        // Rate limiter: 10 requests per second
        RateLimiterConfig rlConfig = RateLimiterConfig.custom()
            .limitForPeriod(10)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .timeoutDuration(Duration.ofMillis(500))
            .build();
        this.rateLimiter = RateLimiter.of("api", rlConfig);
        
        // Bulkhead: max 5 concurrent requests
        BulkheadConfig bulkheadConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(5)
            .maxWaitDuration(Duration.ofMillis(500))
            .build();
        this.bulkhead = Bulkhead.of("api", bulkheadConfig);
        
        setupEventListeners();
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
                throw new RuntimeException("HTTP request failed", e);
            }
        };
        
        // Apply all resilience patterns
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .withFallback(
                Arrays.asList(Exception.class),
                e -> {
                    System.err.println("All resilience patterns failed: " + 
                        e.getMessage());
                    return "Fallback response";
                }
            )
            .decorate()
            .get();
    }
    
    private void setupEventListeners() {
        circuitBreaker.getEventPublisher().onStateTransition(event -> {
            System.out.println("[Circuit Breaker] State: " + 
                event.getStateTransition());
        });
        
        retry.getEventPublisher().onRetry(event -> {
            System.out.println("[Retry] Attempt " + 
                event.getNumberOfRetryAttempts());
        });
    }
    
    public static void main(String[] args) {
        ResilientHttpClient client = new ResilientHttpClient();
        
        // Make requests
        for (int i = 0; i < 20; i++) {
            try {
                String response = client.get("https://api.example.com/data");
                System.out.println("Response: " + response);
            } catch (Exception e) {
                System.err.println("Request failed: " + e.getMessage());
            }
            
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}
```

### Database Connection Pool with Resilience

```java
public class ResilientDatabaseClient {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    private final DataSource dataSource;
    
    public ResilientDatabaseClient(DataSource dataSource) {
        this.dataSource = dataSource;
        
        // Circuit breaker for database
        CircuitBreakerConfig cbConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(5)
            .slidingWindowSize(50)
            .recordExceptions(SQLException.class)
            .build();
        this.circuitBreaker = CircuitBreaker.of("database", cbConfig);
        
        // Retry for transient database errors
        RetryConfig retryConfig = RetryConfig.custom()
            .maxAttempts(2)
            .waitDuration(Duration.ofMillis(100))
            .retryExceptions(SQLTransientException.class)
            .build();
        this.retry = Retry.of("database", retryConfig);
        
        // Bulkhead to limit concurrent database connections
        BulkheadConfig bulkheadConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(20)
            .maxWaitDuration(Duration.ofSeconds(1))
            .build();
        this.bulkhead = Bulkhead.of("database", bulkheadConfig);
    }
    
    public List<User> findAllUsers() {
        Supplier<List<User>> supplier = () -> {
            try (Connection conn = dataSource.getConnection();
                 PreparedStatement stmt = conn.prepareStatement(
                     "SELECT id, name, email FROM users")) {
                
                ResultSet rs = stmt.executeQuery();
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
                throw new RuntimeException("Database query failed", e);
            }
        };
        
        return Decorators.ofSupplier(supplier)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withBulkhead(bulkhead)
            .withFallback(
                Arrays.asList(Exception.class),
                e -> {
                    System.err.println("Database operation failed: " + 
                        e.getMessage());
                    return Collections.emptyList();
                }
            )
            .decorate()
            .get();
    }
    
    public void saveUser(User user) {
        Runnable runnable = () -> {
            try (Connection conn = dataSource.getConnection();
                 PreparedStatement stmt = conn.prepareStatement(
                     "INSERT INTO users (id, name, email) VALUES (?, ?, ?)")) {
                
                stmt.setString(1, user.getId());
                stmt.setString(2, user.getName());
                stmt.setString(3, user.getEmail());
                stmt.executeUpdate();
                
            } catch (SQLException e) {
                throw new RuntimeException("Failed to save user", e);
            }
        };
        
        Decorators.ofRunnable(runnable)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .withBulkhead(bulkhead)
            .decorate()
            .run();
    }
}
```

### Message Queue Producer with Resilience

```java
public class ResilientMessageProducer {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final RateLimiter rateLimiter;
    private final MessageQueue messageQueue;
    
    public ResilientMessageProducer(MessageQueue messageQueue) {
        this.messageQueue = messageQueue;
        
        CircuitBreakerConfig cbConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .build();
        this.circuitBreaker = CircuitBreaker.of("messageQueue", cbConfig);
        
        RetryConfig retryConfig = RetryConfig.custom()
            .maxAttempts(3)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(100),
                2.0
            ))
            .build();
        this.retry = Retry.of("messageQueue", retryConfig);
        
        RateLimiterConfig rlConfig = RateLimiterConfig.custom()
            .limitForPeriod(100)
            .limitRefreshPeriod(Duration.ofSeconds(1))
            .build();
        this.rateLimiter = RateLimiter.of("messageQueue", rlConfig);
    }
    
    public void sendMessage(String topic, String message) {
        Runnable runnable = () -> {
            try {
                messageQueue.send(topic, message);
            } catch (Exception e) {
                throw new RuntimeException("Failed to send message", e);
            }
        };
        
        try {
            Decorators.ofRunnable(runnable)
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .withRateLimiter(rateLimiter)
                .decorate()
                .run();
        } catch (Exception e) {
            // Fallback: store to dead letter queue
            storeToDeadLetterQueue(topic, message, e);
        }
    }
    
    private void storeToDeadLetterQueue(String topic, String message, Exception e) {
        System.err.println("Storing to DLQ: " + topic + " - " + e.getMessage());
        // DLQ implementation
    }
}
```

---

## Conclusion

**Key Benefits of Programmatic Approach:**

1. **No Spring Dependency** - Works in any Java application
2. **Fine-Grained Control** - Explicit configuration per call
3. **Type Safety** - Compile-time checks
4. **Testing** - Easy to test without Spring context
5. **Flexibility** - Mix and match patterns as needed

**Common Patterns:**

```java
// Basic pattern
CircuitBreaker cb = CircuitBreaker.ofDefaults("service");
String result = CircuitBreaker.decorateSupplier(cb, () -> call()).get();

// Combined patterns
Supplier<String> decorated = Decorators.ofSupplier(() -> call())
    .withCircuitBreaker(circuitBreaker)
    .withRetry(retry)
    .withRateLimiter(rateLimiter)
    .withBulkhead(bulkhead)
    .withFallback(List.of(Exception.class), e -> "fallback")
    .decorate();

String result = decorated.get();
```

**Remember**: The programmatic approach gives you full control and works anywhere Java runs!

