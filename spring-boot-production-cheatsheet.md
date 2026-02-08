# 🚀 Spring Boot Production-Ready Cheat Sheet

> **Purpose:** Production-hardened Spring Boot configuration, lifecycle, and operational patterns. Reference before deploying any service to production.
> **Stack context:** Java 21+ / Spring Boot 3.x / Spring Cloud / MongoDB / Kafka / Artemis / Virtual Threads

---

## 📋 Production Readiness Checklist

Before any production deployment, verify:

- [ ] Health checks configured and meaningful
- [ ] Graceful shutdown enabled with appropriate timeout
- [ ] Actuator endpoints secured (not publicly exposed)
- [ ] Structured logging with correlation IDs
- [ ] Metrics exported (Micrometer → Prometheus/Datadog)
- [ ] Connection pools sized and monitored
- [ ] Timeouts on ALL external calls
- [ ] Circuit breakers on all downstream services
- [ ] Profiles correctly separated (dev/staging/prod)
- [ ] Secrets externalized (not in application.yml)
- [ ] Container resource limits (CPU, memory) set
- [ ] JVM tuned for container environment
- [ ] Startup/readiness/liveness probes defined

---

## ⚙️ Pattern 1: Auto-Configuration & Profiles

### Profile Strategy

```
application.yml                ← Shared defaults (all environments)
application-local.yml          ← Local dev (H2, embedded Kafka, debug logging)
application-integration.yml    ← Integration tests (Testcontainers overrides)
application-staging.yml        ← Staging (real infra, reduced resources)
application-prod.yml           ← Production (full config, strict security)
```

### application.yml — Shared Defaults

```yaml
spring:
  application:
    name: payment-service
  
  # ── Jackson ──
  jackson:
    default-property-inclusion: non_null
    deserialization:
      fail-on-unknown-properties: false    # Forward compatibility
    serialization:
      write-dates-as-timestamps: false     # ISO-8601

  # ── Virtual Threads (Java 21+) ──
  threads:
    virtual:
      enabled: true    # ALL request handling on virtual threads

  # ── Lifecycle ──
  lifecycle:
    timeout-per-shutdown-phase: 30s

  main:
    banner-mode: off

# ── Server ──
server:
  port: 8080
  shutdown: graceful
  tomcat:
    connection-timeout: 5s
    keep-alive-timeout: 15s
    max-connections: 8192
    threads:
      max: 200
      min-spare: 10
    accept-count: 100

# ── Management / Actuator ──
management:
  server:
    port: 8081                 # Separate port — not exposed externally
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,configprops
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      show-components: when_authorized
      probes:
        enabled: true          # Enables /actuator/health/liveness and /readiness
    env:
      show-values: when_authorized
  health:
    circuitbreakers:
      enabled: true
    kafka:
      enabled: true
    mongo:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        spring.kafka.listener: true
      minimum-expected-value:
        http.server.requests: 1ms
      maximum-expected-value:
        http.server.requests: 10s
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
      instance: ${HOSTNAME:unknown}
  observations:
    key-values:
      application: ${spring.application.name}

# ── Logging ──
logging:
  pattern:
    console: >
      %d{ISO8601} %5level [${spring.application.name},%X{traceId:-},%X{spanId:-}]
      [%thread] %logger{36} - %msg%n
  level:
    root: INFO
    com.example.payment: INFO
    org.springframework.kafka: WARN
    org.mongodb.driver: WARN
    org.apache.kafka: WARN
```

### application-prod.yml — Production Overrides

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}
      auto-index-creation: false     # Indexes managed by migration scripts in prod

  kafka:
    bootstrap-servers: ${KAFKA_BROKERS}
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-256
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${KAFKA_USER}" password="${KAFKA_PASSWORD}";

logging:
  level:
    root: WARN
    com.example.payment: INFO
  # Structured JSON for log aggregation
  pattern:
    console: >
      {"timestamp":"%d{ISO8601}","level":"%level","service":"${spring.application.name}",
       "traceId":"%X{traceId:-}","spanId":"%X{spanId:-}","thread":"%thread",
       "logger":"%logger{36}","message":"%msg"}%n

management:
  endpoint:
    health:
      show-details: never       # Don't leak infra details in prod
    env:
      enabled: false            # Disable env endpoint in prod
```

---

## 🩺 Pattern 2: Health Checks & Probes

### Kubernetes Probe Mapping

```
STARTUP PROBE    → /actuator/health/liveness
  When: Container just started
  Purpose: "Has the app finished initializing?"
  Failure: Kubernetes RESTARTS the container
  Config: initialDelaySeconds: 10, periodSeconds: 5, failureThreshold: 30

LIVENESS PROBE   → /actuator/health/liveness
  When: App is running
  Purpose: "Is the app stuck/deadlocked?"
  Failure: Kubernetes RESTARTS the container
  Config: periodSeconds: 10, failureThreshold: 3
  ⚠️ NEVER include external dependency checks here

READINESS PROBE  → /actuator/health/readiness
  When: App is running
  Purpose: "Can this instance handle traffic?"
  Failure: Kubernetes STOPS sending traffic (doesn't restart)
  Config: periodSeconds: 5, failureThreshold: 3
  ✅ Include: DB connectivity, Kafka connectivity, circuit breaker state
```

### Custom Health Indicators

```java
// ── Critical dependency — affects readiness ──
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry circuitBreakerRegistry;

    @Override
    public Health health() {
        var breaker = circuitBreakerRegistry.circuitBreaker("payment-gateway");
        return switch (breaker.getState()) {
            case CLOSED, HALF_OPEN -> Health.up()
                .withDetail("state", breaker.getState())
                .withDetail("failureRate", breaker.getMetrics().getFailureRate())
                .build();
            case OPEN -> Health.down()
                .withDetail("state", "OPEN")
                .withDetail("failureRate", breaker.getMetrics().getFailureRate())
                .withDetail("waitDuration", "Waiting for half-open transition")
                .build();
            default -> Health.unknown().build();
        };
    }
}

// ── Readiness group — controls traffic routing ──
@Component
public class KafkaConsumerHealthIndicator implements HealthIndicator {

    private final KafkaListenerEndpointRegistry registry;

    @Override
    public Health health() {
        boolean allRunning = registry.getListenerContainers().stream()
            .allMatch(MessageListenerContainer::isRunning);

        return allRunning
            ? Health.up().withDetail("listeners", registry.getListenerContainerIds()).build()
            : Health.down().withDetail("stoppedListeners",
                registry.getListenerContainers().stream()
                    .filter(c -> !c.isRunning())
                    .map(MessageListenerContainer::getListenerId)
                    .toList()).build();
    }
}
```

### Health Group Configuration

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState      # Only JVM-level — never external deps
        readiness:
          include: readinessState, mongo, kafka, paymentGateway
```

---

## 🛑 Pattern 3: Graceful Shutdown

### Shutdown Sequence

```
SIGTERM received
     │
     ▼
┌─────────────────────────────────┐
│ 1. Readiness probe → DOWN       │  ← Stop receiving new traffic
│    (Kubernetes stops routing)   │
├─────────────────────────────────┤
│ 2. Pre-shutdown delay (5s)      │  ← Wait for in-flight LB connections
├─────────────────────────────────┤
│ 3. Stop Kafka consumers         │  ← Stop polling, finish current batch
│    Commit offsets               │
├─────────────────────────────────┤
│ 4. Stop JMS listeners           │  ← Stop receiving, ack pending messages
├─────────────────────────────────┤
│ 5. Complete HTTP requests       │  ← Finish in-flight requests (up to timeout)
├─────────────────────────────────┤
│ 6. Flush outbox                 │  ← Publish any remaining outbox events
├─────────────────────────────────┤
│ 7. Close connections            │  ← DB pools, Kafka producers, JMS connections
├─────────────────────────────────┤
│ 8. Destroy Spring context       │  ← @PreDestroy methods
├─────────────────────────────────┤
│ 9. JVM shutdown                 │
└─────────────────────────────────┘
    Total budget: 30 seconds (spring.lifecycle.timeout-per-shutdown-phase)
```

### Shutdown Configuration & Hooks

```java
@Configuration
public class GracefulShutdownConfig {

    // Pre-shutdown delay — allows load balancer to drain
    @Bean
    public ApplicationListener<ContextClosedEvent> preShutdownDelay() {
        return event -> {
            log.info("Shutdown initiated — waiting 5s for LB drain");
            try { Thread.sleep(5000); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };
    }
}

@Component
public class KafkaShutdownHandler {

    private final KafkaListenerEndpointRegistry registry;

    @PreDestroy
    public void shutdown() {
        log.info("Stopping Kafka listeners gracefully...");
        registry.getListenerContainers().forEach(container -> {
            log.info("Stopping listener: {}", container.getListenerId());
            container.stop();
        });
        log.info("All Kafka listeners stopped");
    }
}

@Component
public class OutboxFlushOnShutdown {

    private final OutboxPoller outboxPoller;

    @PreDestroy
    public void flushOutbox() {
        log.info("Flushing remaining outbox events...");
        outboxPoller.pollAndPublish(); // One final flush
        log.info("Outbox flush complete");
    }
}
```

---

## 🧵 Pattern 4: Virtual Threads Configuration

### When Virtual Threads Help

```
✅ USE virtual threads for:
  • HTTP request handling (I/O-bound)
  • Database queries
  • Kafka consuming/producing
  • REST client calls
  • JMS messaging
  • File I/O

❌ AVOID virtual threads for:
  • CPU-intensive computation (hashing, encryption, serialization)
  • Operations using synchronized blocks (pin the carrier thread)
  • Code using ThreadLocal extensively (memory overhead per virtual thread)
```

### Complete Virtual Thread Setup

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true     # Tomcat + Kafka + JMS + @Async all use virtual threads

  task:
    execution:
      thread-name-prefix: vt-
    scheduling:
      thread-name-prefix: vt-sched-
```

```java
@Configuration
public class VirtualThreadConfig {

    // Custom executor for @Async and CompletableFuture
    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }

    // Kafka consumer with virtual threads
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> cf) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(cf);
        factory.getContainerProperties().setListenerTaskExecutor(
            new SimpleAsyncTaskExecutor("kafka-vt-"));  // Virtual threads in Spring Boot 3.2+
        return factory;
    }
}
```

### Avoiding Virtual Thread Pitfalls

```java
// ❌ synchronized blocks PIN the carrier thread
public synchronized void process(Payment payment) { ... }

// ✅ Use ReentrantLock instead
private final ReentrantLock lock = new ReentrantLock();
public void process(Payment payment) {
    lock.lock();
    try { ... } finally { lock.unlock(); }
}

// ❌ ThreadLocal with large data — one per virtual thread
private static final ThreadLocal<HeavyObject> cache = ThreadLocal.withInitial(HeavyObject::new);

// ✅ Use ScopedValue (Java 24 preview) or pass as parameter
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();
```

---

## 📊 Pattern 5: Metrics & Observability

### Micrometer Custom Metrics

```java
@Component
public class PaymentMetrics {

    private final MeterRegistry registry;

    // ── Counter — events that happened ──
    public void recordPaymentProcessed(PaymentType type, String outcome) {
        registry.counter("payment.processed",
            "type", type.name(),
            "outcome", outcome     // "success", "failed", "rejected"
        ).increment();
    }

    // ── Timer — how long things take ──
    public <T> T timeGatewayCall(String gateway, Supplier<T> action) {
        return registry.timer("payment.gateway.duration",
            "gateway", gateway
        ).record(action);
    }

    // ── Gauge — current state ──
    @PostConstruct
    public void registerGauges() {
        Gauge.builder("payment.outbox.pending", outboxRepository,
            repo -> repo.countByStatus(OutboxStatus.PENDING))
            .register(registry);

        Gauge.builder("payment.circuit_breaker.state",
            circuitBreakerRegistry.circuitBreaker("payment-gateway"),
            cb -> switch (cb.getState()) {
                case CLOSED -> 0;
                case HALF_OPEN -> 1;
                case OPEN -> 2;
                default -> -1;
            })
            .tag("service", "payment-gateway")
            .register(registry);
    }

    // ── Distribution Summary — value distributions ──
    public void recordPaymentAmount(BigDecimal amount, String currency) {
        registry.summary("payment.amount",
            "currency", currency
        ).record(amount.doubleValue());
    }
}
```

### Structured Logging with Context

```java
@Component
public class StructuredLoggingFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String traceId = exchange.getRequest().getHeaders()
            .getFirst("X-Trace-Id");
        if (traceId == null) traceId = UUID.randomUUID().toString();

        MDC.put("traceId", traceId);
        MDC.put("method", exchange.getRequest().getMethod().name());
        MDC.put("path", exchange.getRequest().getPath().value());
        MDC.put("clientIp", Objects.toString(
            exchange.getRequest().getRemoteAddress(), "unknown"));

        return chain.filter(exchange)
            .doFinally(signal -> MDC.clear());
    }
}

// Logback JSON encoder config (logback-spring.xml)
// <encoder class="net.logstash.logback.encoder.LogstashEncoder">
//   <includeMdcKeyName>traceId</includeMdcKeyName>
//   <includeMdcKeyName>spanId</includeMdcKeyName>
//   <includeMdcKeyName>correlationId</includeMdcKeyName>
// </encoder>
```

---

## 🔌 Pattern 6: Connection Pool Configuration

### MongoDB Connection Pool

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}
      # Connection pool settings (in URI or MongoClientSettings)
      # mongodb://host:27017/db?maxPoolSize=50&minPoolSize=10
      #   &maxIdleTimeMS=60000&waitQueueTimeoutMS=5000
      #   &connectTimeoutMS=3000&socketTimeoutMS=10000
```

```java
@Configuration
public class MongoConfig {

    @Bean
    public MongoClientSettings mongoClientSettings() {
        return MongoClientSettings.builder()
            .applyConnectionString(new ConnectionString(mongoUri))
            .applyToConnectionPoolSettings(pool -> pool
                .maxSize(50)                                    // Max connections
                .minSize(10)                                    // Keep warm
                .maxWaitTime(5, TimeUnit.SECONDS)               // Wait for connection
                .maxConnectionIdleTime(60, TimeUnit.SECONDS)    // Close idle connections
                .maxConnectionLifeTime(30, TimeUnit.MINUTES))   // Rotate connections
            .applyToSocketSettings(socket -> socket
                .connectTimeout(3, TimeUnit.SECONDS)            // TCP connect timeout
                .readTimeout(10, TimeUnit.SECONDS))             // Socket read timeout
            .applyToClusterSettings(cluster -> cluster
                .serverSelectionTimeout(5, TimeUnit.SECONDS))   // Time to find a server
            .build();
    }
}
```

### Connection Pool Sizing Formula

```
Pool Size Formula (per instance):
  maxPoolSize = (concurrent_requests × avg_queries_per_request × avg_query_duration_seconds)
                × safety_factor

Example:
  100 concurrent requests × 3 queries each × 0.05s avg = 15 connections needed
  With 1.5x safety factor = ~23 → round to 25

Multiple instances:
  If 4 instances: each gets maxPoolSize = 25
  Total connections to MongoDB = 100
  Ensure MongoDB maxIncomingConnections > total
```

---

## 🔒 Pattern 7: Security Configuration

### Actuator Security

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                // Public health probes (for Kubernetes)
                .requestMatchers("/actuator/health/liveness").permitAll()
                .requestMatchers("/actuator/health/readiness").permitAll()
                // Protected actuator endpoints
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                // API endpoints
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable())  // Disable for API-only service
            .build();
    }
}
```

### Secrets Management

```yaml
# ❌ NEVER — secrets in application.yml
spring.data.mongodb.uri: mongodb://admin:P@ssw0rd@host:27017/db

# ✅ Environment variables (Kubernetes Secrets → env vars)
spring.data.mongodb.uri: ${MONGODB_URI}

# ✅ Spring Cloud Config + Vault
spring:
  cloud:
    config:
      uri: ${CONFIG_SERVER_URI}
  config:
    import: vault://secret/payment-service
```

---

## 🐳 Pattern 8: Container & JVM Tuning

### Dockerfile — Production

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S app && adduser -S app -G app
WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

# JVM container-aware settings
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom \
  -Dspring.jmx.enabled=false"

USER app
EXPOSE 8080 8081

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8081/actuator/health/liveness || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 45   # > spring.lifecycle.timeout-per-shutdown-phase
      containers:
        - name: payment-service
          image: payment-service:latest
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          ports:
            - containerPort: 8080     # Application
            - containerPort: 8081     # Management
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: mongodb-uri
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30      # 10 + (5 × 30) = up to 160s to start
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            periodSeconds: 5
            failureThreshold: 3
```

### GC Selection Guide

| GC | Best For | Flag | Heap Size |
|----|----------|------|-----------|
| **ZGC (Generational)** | Low latency, large heaps | `-XX:+UseZGC -XX:+ZGenerational` | > 512MB |
| **G1GC** | Balanced throughput/latency | `-XX:+UseG1GC` | 256MB - 4GB |
| **Serial** | Tiny containers, short-lived | `-XX:+UseSerialGC` | < 256MB |
| **Shenandoah** | Low latency alternative to ZGC | `-XX:+UseShenandoahGC` | > 512MB |

---

## 🔄 Pattern 9: Configuration Properties — Type-Safe

```java
// ── Type-safe, validated, immutable configuration ──
@ConfigurationProperties(prefix = "payment")
@Validated
public record PaymentProperties(
    @NotNull GatewayProperties gateway,
    @NotNull RetryProperties retry,
    @NotNull OutboxProperties outbox
) {
    public record GatewayProperties(
        @NotBlank String baseUrl,
        @Positive Duration connectTimeout,
        @Positive Duration readTimeout,
        @Positive int maxConnections
    ) {
        public GatewayProperties {
            if (connectTimeout == null) connectTimeout = Duration.ofSeconds(3);
            if (readTimeout == null) readTimeout = Duration.ofSeconds(10);
            if (maxConnections <= 0) maxConnections = 50;
        }
    }

    public record RetryProperties(
        @Min(1) int maxAttempts,
        @Positive Duration initialDelay,
        double multiplier
    ) {
        public RetryProperties {
            if (maxAttempts <= 0) maxAttempts = 3;
            if (initialDelay == null) initialDelay = Duration.ofMillis(500);
            if (multiplier <= 0) multiplier = 2.0;
        }
    }

    public record OutboxProperties(
        @Positive Duration pollInterval,
        @Positive int batchSize,
        @Min(1) int maxAttempts,
        @Positive Duration cleanupRetention
    ) {}
}

// application.yml
// payment:
//   gateway:
//     base-url: https://gateway.example.com
//     connect-timeout: 3s
//     read-timeout: 10s
//     max-connections: 50
//   retry:
//     max-attempts: 3
//     initial-delay: 500ms
//     multiplier: 2.0
//   outbox:
//     poll-interval: 100ms
//     batch-size: 100
//     max-attempts: 5
//     cleanup-retention: 7d
```

---

## 💡 Golden Rules of Spring Boot Production

```
1.  SEPARATE management port — actuator on 8081, app on 8080. Never expose actuator publicly.
2.  GRACEFUL SHUTDOWN is mandatory — set server.shutdown=graceful and lifecycle timeout.
3.  VIRTUAL THREADS for I/O — one property to enable, dramatic throughput improvement.
4.  HEALTH GROUPS matter — liveness = JVM only, readiness = external deps. Mix them up = cascading restarts.
5.  STRUCTURED JSON LOGGING in prod — plain text is for dev. Log aggregation needs JSON.
6.  TYPE-SAFE CONFIG with records — no more magic strings and @Value scattered everywhere.
7.  CONTAINER-AWARE JVM — always UseContainerSupport + MaxRAMPercentage, never fixed -Xmx in containers.
8.  TIMEOUTS ON EVERYTHING — HTTP clients, DB connections, Kafka producers. Unbounded waits = outages.
9.  EXTERNALIZE SECRETS — env vars or Vault, never in config files, never in Docker images.
10. terminationGracePeriodSeconds > shutdown timeout — give Spring time to drain before Kubernetes kills it.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.2+ / Kubernetes / Docker*
