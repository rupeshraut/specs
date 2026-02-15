# Spring Boot Production Mastery

A comprehensive guide to configuring, deploying, and operating Spring Boot applications in production with enterprise-grade patterns.

---

## Table of Contents

1. [Production Readiness](#production-readiness)
2. [Configuration Management](#configuration-management)
3. [Application Lifecycle](#application-lifecycle)
4. [Actuator Deep Dive](#actuator-deep-dive)
5. [Graceful Shutdown](#graceful-shutdown)
6. [Startup Optimization](#startup-optimization)
7. [Error Handling](#error-handling)
8. [Async Processing](#async-processing)
9. [Caching](#caching)
10. [Database Connection Pooling](#database-connection-pooling)
11. [Production Monitoring](#production-monitoring)
12. [Security Hardening](#security-hardening)

---

## Production Readiness

### Production Checklist

```yaml
# application-production.yml

spring:
  application:
    name: payment-service

  # 1. Externalize configuration
  config:
    import: "optional:configserver:http://config-server:8888"

  # 2. Proper logging
  output:
    ansi:
      enabled: never

logging:
  level:
    root: INFO
    com.company: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
  file:
    name: /var/log/payment-service/application.log
    max-size: 100MB
    max-history: 30

# 3. Actuator for monitoring
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
  metrics:
    export:
      prometheus:
        enabled: true

# 4. Server configuration
server:
  port: 8080
  shutdown: graceful
  tomcat:
    threads:
      max: 200
      min-spare: 10
    connection-timeout: 20s
    max-connections: 8192

# 5. Database pooling
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

# 6. Error handling
server:
  error:
    include-stacktrace: never
    include-message: always
```

---

## Configuration Management

### Environment-Specific Configuration

```yaml
# application.yml (defaults)
spring:
  application:
    name: payment-service

app:
  features:
    new-payment-flow: false
  retry:
    max-attempts: 3

---
# application-development.yml
spring:
  config:
    activate:
      on-profile: development

logging:
  level:
    root: DEBUG
    com.company: DEBUG

app:
  features:
    new-payment-flow: true

---
# application-production.yml
spring:
  config:
    activate:
      on-profile: production

logging:
  level:
    root: INFO
    com.company: INFO

app:
  features:
    new-payment-flow: false
```

### Configuration Properties

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class ApplicationProperties {

    @NotNull
    private Features features = new Features();

    @NotNull
    private Retry retry = new Retry();

    @NotNull
    private Integration integration = new Integration();

    public static class Features {
        private boolean newPaymentFlow = false;
        private boolean emailNotifications = true;

        // Getters/setters
    }

    public static class Retry {
        @Min(1)
        @Max(10)
        private int maxAttempts = 3;

        @Min(100)
        private long backoffMs = 1000;

        // Getters/setters
    }

    public static class Integration {
        @NotBlank
        private String paymentGatewayUrl;

        @Min(1000)
        private int timeoutMs = 5000;

        // Getters/setters
    }
}

// Enable in configuration
@Configuration
@EnableConfigurationProperties(ApplicationProperties.class)
public class AppConfig {
}
```

---

## Application Lifecycle

### Startup Events

```java
@Component
@Slf4j
public class StartupListener {

    @EventListener(ApplicationStartedEvent.class)
    public void onApplicationStarted() {
        log.info("Application started - initializing resources");
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        log.info("Application ready to serve traffic");
        // Warm up caches
        // Establish connections
        // Register with service discovery
    }

    @EventListener(ApplicationFailedEvent.class)
    public void onApplicationFailed(ApplicationFailedEvent event) {
        log.error("Application failed to start", event.getException());
    }
}
```

### Shutdown Events

```java
@Component
@Slf4j
public class ShutdownListener {

    private final ExecutorService executorService;

    @PreDestroy
    public void onShutdown() {
        log.info("Application shutting down - cleaning up resources");

        // Stop accepting new requests
        // Complete in-flight requests
        executorService.shutdown();
        try {
            if (!executorService.awaitTermination(30, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        } catch (InterruptedException e) {
            executorService.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## Actuator Deep Dive

### Custom Health Indicator

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final WebClient paymentGatewayClient;

    @Override
    public Health health() {
        try {
            // Check payment gateway
            HttpStatus status = paymentGatewayClient
                .get()
                .uri("/health")
                .retrieve()
                .toBodilessEntity()
                .timeout(Duration.ofSeconds(3))
                .block()
                .getStatusCode();

            if (status.is2xxSuccessful()) {
                return Health.up()
                    .withDetail("gateway", "reachable")
                    .withDetail("responseTime", "45ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("gateway", "unhealthy")
                    .withDetail("status", status.value())
                    .build();
            }

        } catch (Exception e) {
            return Health.down()
                .withDetail("gateway", "unreachable")
                .withException(e)
                .build();
        }
    }
}
```

### Custom Metrics

```java
@Component
public class PaymentMetrics {

    private final Counter paymentsProcessed;
    private final Timer paymentProcessingTime;
    private final Gauge activePayments;

    public PaymentMetrics(MeterRegistry registry) {
        this.paymentsProcessed = Counter.builder("payments.processed")
            .description("Total payments processed")
            .tags("service", "payment")
            .register(registry);

        this.paymentProcessingTime = Timer.builder("payment.processing.time")
            .description("Payment processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.activePayments = Gauge.builder("payments.active",
                this::getActivePaymentCount)
            .description("Currently active payments")
            .register(registry);
    }

    public void recordPaymentProcessed() {
        paymentsProcessed.increment();
    }

    public void recordProcessingTime(Duration duration) {
        paymentProcessingTime.record(duration);
    }

    private long getActivePaymentCount() {
        // Return current count from cache/db
        return 0L;
    }
}
```

### Info Endpoint

```yaml
# application.yml
info:
  app:
    name: ${spring.application.name}
    description: Payment Processing Service
    version: @project.version@
    encoding: @project.build.sourceEncoding@
    java:
      version: @java.version@

management:
  info:
    env:
      enabled: true
    git:
      enabled: true
      mode: full
```

```java
@Component
public class CustomInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("uptime", getUptime())
               .withDetail("environment", getEnvironment())
               .withDetail("custom", Map.of(
                   "feature-flags", getFeatureFlags(),
                   "integrations", getIntegrationStatus()
               ));
    }
}
```

---

## Graceful Shutdown

### Configuration

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### Implementation

```java
@Configuration
public class GracefulShutdownConfig {

    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }
}

@Slf4j
public class GracefulShutdown implements TomcatConnectorCustomizer,
                                        ApplicationListener<ContextClosedEvent> {

    private volatile Connector connector;
    private static final int SHUTDOWN_TIMEOUT_SECONDS = 30;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        if (connector == null) {
            return;
        }

        // Pause connector - stop accepting new requests
        connector.pause();

        log.info("Paused connector, waiting for {} seconds for requests to complete",
            SHUTDOWN_TIMEOUT_SECONDS);

        Executor executor = connector.getProtocolHandler().getExecutor();
        if (executor instanceof ThreadPoolExecutor) {
            try {
                ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                threadPoolExecutor.shutdown();

                if (!threadPoolExecutor.awaitTermination(
                        SHUTDOWN_TIMEOUT_SECONDS, TimeUnit.SECONDS)) {
                    log.warn("Tomcat thread pool did not shut down gracefully, forcing shutdown");
                    threadPoolExecutor.shutdownNow();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

## Startup Optimization

### Lazy Initialization

```yaml
spring:
  main:
    lazy-initialization: true # Global lazy loading

  # Or selective
  lazy-initialization: false

# Individual beans
@Configuration
public class AppConfig {

    @Bean
    @Lazy
    public ExpensiveService expensiveService() {
        return new ExpensiveService();
    }
}
```

### Component Scanning Optimization

```java
@SpringBootApplication(
    scanBasePackages = "com.company.payment" // Specific package
)
@EnableAutoConfiguration(
    exclude = {
        DataSourceAutoConfiguration.class,
        MongoAutoConfiguration.class
    }
)
public class PaymentApplication {
}
```

### AOT Compilation (Spring Native)

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

---

## Error Handling

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(PaymentNotFoundException.class)
    public ResponseEntity<ErrorResponse> handlePaymentNotFound(
            PaymentNotFoundException ex) {

        log.warn("Payment not found: {}", ex.getPaymentId());

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Payment Not Found")
            .message(ex.getMessage())
            .path(getCurrentRequest())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(PaymentProcessingException.class)
    public ResponseEntity<ErrorResponse> handlePaymentProcessing(
            PaymentProcessingException ex) {

        log.error("Payment processing failed", ex);

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Payment Processing Failed")
            .message("Unable to process payment")
            .traceId(MDC.get("traceId"))
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {

        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> error.getDefaultMessage() != null ?
                    error.getDefaultMessage() : ""
            ));

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Invalid request parameters")
            .validationErrors(errors)
            .build();

        return ResponseEntity.badRequest().body(error);
    }
}
```

---

## Async Processing

### Thread Pool Configuration

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy()
        );

        executor.initialize();
        return executor;
    }

    @Bean(name = "longRunningExecutor")
    public Executor longRunningExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("long-running-");
        executor.setKeepAliveSeconds(60);

        executor.initialize();
        return executor;
    }
}
```

### Async Processing

```java
@Service
public class PaymentNotificationService {

    @Async("taskExecutor")
    public CompletableFuture<Void> sendEmail(PaymentEvent event) {
        try {
            emailService.send(event.getUserEmail(),
                "Payment Processed",
                createEmailBody(event));

            return CompletableFuture.completedFuture(null);

        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }

    @Async("longRunningExecutor")
    public CompletableFuture<Report> generateReport(String userId) {
        // Long-running operation
        Report report = reportGenerator.generate(userId);
        return CompletableFuture.completedFuture(report);
    }
}
```

---

## Caching

### Cache Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager(
            "payments", "users", "configurations"
        );

        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats()
        );

        return cacheManager;
    }
}
```

### Using Cache

```java
@Service
public class PaymentService {

    @Cacheable(value = "payments", key = "#id")
    public Payment findById(String id) {
        return paymentRepository.findById(id)
            .orElseThrow(() -> new PaymentNotFoundException(id));
    }

    @CachePut(value = "payments", key = "#result.id")
    public Payment save(Payment payment) {
        return paymentRepository.save(payment);
    }

    @CacheEvict(value = "payments", key = "#id")
    public void delete(String id) {
        paymentRepository.deleteById(id);
    }

    @CacheEvict(value = "payments", allEntries = true)
    public void clearCache() {
        // Clear all payment cache
    }
}
```

---

## Production Best Practices

### ✅ DO

- Use profiles for environment-specific config
- Enable actuator health checks
- Configure graceful shutdown
- Use connection pooling
- Implement proper error handling
- Enable metrics and monitoring
- Use async processing where appropriate
- Configure proper logging
- Externalize configuration
- Test with production-like config

### ❌ DON'T

- Hardcode configuration values
- Expose stack traces in production
- Use default thread pool sizes
- Ignore startup/shutdown lifecycle
- Skip health check implementation
- Use blocking operations in async methods
- Expose all actuator endpoints
- Use development config in production
- Ignore connection pool tuning
- Skip graceful shutdown configuration

---

*This guide provides comprehensive patterns for running Spring Boot applications in production. Proper configuration and lifecycle management are critical for reliable operations.*
