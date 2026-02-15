# Observability Engineering

A comprehensive guide to implementing production-grade observability with metrics, logging, tracing, and alerting for distributed Java systems.

---

## Table of Contents

1. [Observability Fundamentals](#observability-fundamentals)
2. [The Three Pillars](#the-three-pillars)
3. [Metrics with Micrometer](#metrics-with-micrometer)
4. [Distributed Tracing](#distributed-tracing)
5. [Structured Logging](#structured-logging)
6. [Correlation IDs](#correlation-ids)
7. [SLI, SLO, SLA](#sli-slo-sla)
8. [Alerting Strategy](#alerting-strategy)
9. [Dashboards](#dashboards)
10. [OpenTelemetry](#opentelemetry)
11. [Production Patterns](#production-patterns)

---

## Observability Fundamentals

### Monitoring vs Observability

```
Monitoring: "Is the system up?"
- Known knowns (predefined metrics)
- Dashboard-driven
- Reactive

Observability: "Why is the system behaving this way?"
- Unknown unknowns (exploratory)
- Data-driven
- Proactive

Goal: Answer ANY question about system behavior
      without deploying new code
```

### The Cardinality Problem

```
Low Cardinality (Good for Metrics):
- service_name: payment-service
- http_status: 200, 404, 500
- endpoint: /api/payments

High Cardinality (Bad for Metrics):
- user_id: millions of values
- payment_id: unique per transaction
- timestamp: infinite values

Solution: Use high-cardinality data in traces/logs, not metrics
```

---

## The Three Pillars

### 1. Metrics

```
What: Numeric measurements over time
When: System-level health, trends, alerts
Examples:
  - Request rate
  - Error rate
  - Response time (p50, p95, p99)
  - CPU, memory usage
  - Queue depth
```

### 2. Logs

```
What: Discrete events with context
When: Debugging, auditing, investigation
Examples:
  - "Payment 12345 processed"
  - "Database connection failed"
  - "User login attempt"
```

### 3. Traces

```
What: Request journey through distributed system
When: Understanding latency, dependencies
Examples:
  - API Gateway → Auth → Payment → Database
  - Total: 250ms (API: 10ms, Auth: 50ms, Payment: 190ms)
```

---

## Metrics with Micrometer

### Basic Metrics

```java
@Service
public class PaymentService {

    private final MeterRegistry meterRegistry;
    private final Counter paymentsProcessed;
    private final Timer paymentProcessingTime;

    public PaymentService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        this.paymentsProcessed = Counter.builder("payments.processed")
            .description("Total number of payments processed")
            .tag("service", "payment")
            .register(meterRegistry);

        this.paymentProcessingTime = Timer.builder("payment.processing.time")
            .description("Time taken to process a payment")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
    }

    public Payment processPayment(PaymentRequest request) {
        return paymentProcessingTime.record(() -> {
            Payment payment = doProcessPayment(request);

            paymentsProcessed.increment();

            // Tag with status
            meterRegistry.counter("payments.by.status",
                "status", payment.getStatus().toString()
            ).increment();

            return payment;
        });
    }
}
```

### Custom Metrics

```java
@Component
public class BusinessMetrics {

    private final MeterRegistry registry;

    // Gauge - current value
    public void registerQueueSize(Queue<Message> queue) {
        Gauge.builder("message.queue.size", queue, Queue::size)
            .description("Current message queue size")
            .tag("queue", "payment-processing")
            .register(registry);
    }

    // Distribution summary - percentiles
    public void recordPaymentAmount(BigDecimal amount) {
        DistributionSummary.builder("payment.amount")
            .description("Payment amount distribution")
            .baseUnit("USD")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry)
            .record(amount.doubleValue());
    }

    // Counter with tags
    public void recordPaymentMethod(String method, boolean success) {
        Counter.builder("payments.by.method")
            .tag("method", method)
            .tag("success", String.valueOf(success))
            .register(registry)
            .increment();
    }

    // Timer with percentiles
    public void timeOperation(String operation, Runnable task) {
        Timer.builder("operation.duration")
            .tag("operation", operation)
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry)
            .record(task);
    }
}
```

### Prometheus Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 10ms,50ms,100ms,200ms,500ms,1s,2s
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

---

## Distributed Tracing

### OpenTelemetry with Spring Boot

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 0.1 (10%) in prod

  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces
```

### Manual Span Creation

```java
@Service
public class PaymentService {

    private final Tracer tracer;

    public Payment processPayment(PaymentRequest request) {
        Span span = tracer.spanBuilder("processPayment")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Add attributes
            span.setAttribute("payment.id", request.getPaymentId());
            span.setAttribute("payment.amount", request.getAmount().doubleValue());
            span.setAttribute("payment.currency", request.getCurrency());

            Payment payment = doProcessPayment(request);

            span.setAttribute("payment.status", payment.getStatus().toString());
            span.setStatus(StatusCode.OK);

            return payment;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;

        } finally {
            span.end();
        }
    }

    private Payment doProcessPayment(PaymentRequest request) {
        // Create child span
        Span childSpan = tracer.spanBuilder("validatePayment")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();

        try (Scope scope = childSpan.makeCurrent()) {
            validatePayment(request);
            childSpan.setStatus(StatusCode.OK);
        } finally {
            childSpan.end();
        }

        // Process payment...
    }
}
```

### Trace Context Propagation

```java
@Configuration
public class KafkaTracingConfig {

    @Bean
    public ProducerFactory<String, PaymentEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        // ... kafka config

        // Add trace context to headers
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, PaymentEvent> kafkaTemplate(
            ProducerFactory<String, PaymentEvent> factory,
            Tracer tracer) {

        KafkaTemplate<String, PaymentEvent> template = new KafkaTemplate<>(factory);

        template.setProducerInterceptor(new ProducerInterceptor<>() {
            @Override
            public ProducerRecord<String, PaymentEvent> onSend(
                    ProducerRecord<String, PaymentEvent> record) {

                // Inject trace context into headers
                Span span = tracer.spanBuilder("kafka.send")
                    .setSpanKind(SpanKind.PRODUCER)
                    .startSpan();

                Headers headers = record.headers();
                W3CTraceContextPropagator.getInstance()
                    .inject(Context.current(), headers, HeadersSetter.INSTANCE);

                span.end();

                return record;
            }

            // ... other methods
        });

        return template;
    }
}
```

---

## Structured Logging

### Logback Configuration

```xml
<!-- logback-spring.xml -->
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- Console output for local development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>tenantId</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- JSON output for production -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.json</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.json.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"application":"payment-service"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

### Structured Logging in Code

```java
@Slf4j
@Service
public class PaymentService {

    public Payment processPayment(PaymentRequest request) {
        // Structured logging with context
        log.info("Processing payment",
            kv("paymentId", request.getPaymentId()),
            kv("amount", request.getAmount()),
            kv("currency", request.getCurrency()),
            kv("userId", request.getUserId())
        );

        try {
            Payment payment = doProcessPayment(request);

            log.info("Payment processed successfully",
                kv("paymentId", payment.getId()),
                kv("status", payment.getStatus()),
                kv("processingTimeMs", payment.getProcessingTime())
            );

            return payment;

        } catch (InsufficientFundsException e) {
            log.warn("Payment failed - insufficient funds",
                kv("paymentId", request.getPaymentId()),
                kv("requestedAmount", request.getAmount()),
                kv("availableBalance", e.getAvailableBalance())
            );
            throw e;

        } catch (Exception e) {
            log.error("Payment processing failed",
                kv("paymentId", request.getPaymentId()),
                kv("errorType", e.getClass().getSimpleName()),
                e
            );
            throw e;
        }
    }

    // Helper for key-value pairs
    private static LogstashMarker kv(String key, Object value) {
        return Markers.append(key, value);
    }
}
```

---

## Correlation IDs

### Filter for Request Tracking

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String CORRELATION_ID_MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain)
            throws ServletException, IOException {

        String correlationId = request.getHeader(CORRELATION_ID_HEADER);

        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_MDC_KEY);
        }
    }
}
```

### Propagating Correlation ID

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();

        restTemplate.setInterceptors(List.of(
            (request, body, execution) -> {
                String correlationId = MDC.get("correlationId");
                if (correlationId != null) {
                    request.getHeaders().set("X-Correlation-ID", correlationId);
                }

                String traceId = MDC.get("traceId");
                if (traceId != null) {
                    request.getHeaders().set("X-Trace-ID", traceId);
                }

                return execution.execute(request, body);
            }
        ));

        return restTemplate;
    }
}
```

---

## SLI, SLO, SLA

### Service Level Indicators (SLI)

```java
@Component
public class PaymentSLI {

    private final MeterRegistry registry;

    // Availability SLI
    public void recordAvailability(boolean success) {
        Counter.builder("sli.availability")
            .tag("success", String.valueOf(success))
            .register(registry)
            .increment();
    }

    // Latency SLI
    public void recordLatency(long durationMs) {
        Timer.builder("sli.latency")
            .publishPercentiles(0.95, 0.99)
            .register(registry)
            .record(Duration.ofMillis(durationMs));
    }

    // Error rate SLI
    public void recordError(String errorType) {
        Counter.builder("sli.errors")
            .tag("errorType", errorType)
            .register(registry)
            .increment();
    }
}
```

### Service Level Objectives (SLO)

```
Payment Service SLOs:

1. Availability
   - Target: 99.9% uptime
   - Measurement: Successful requests / Total requests
   - Window: 30 days

2. Latency
   - Target: 95% of requests < 200ms
   - Measurement: p95 response time
   - Window: 1 day

3. Error Rate
   - Target: < 0.1% errors
   - Measurement: Failed requests / Total requests
   - Window: 1 hour

4. Data Consistency
   - Target: 99.99% eventual consistency within 5s
   - Measurement: Reconciliation checks
   - Window: 1 day
```

### PromQL for SLO Monitoring

```promql
# Availability SLO (99.9%)
sum(rate(http_server_requests_seconds_count{status!~"5.."}[30d]))
/
sum(rate(http_server_requests_seconds_count[30d]))

# Latency SLO (p95 < 200ms)
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket[1d])) by (le)
) < 0.2

# Error budget remaining
1 - (
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[30d]))
  /
  sum(rate(http_server_requests_seconds_count[30d]))
) / 0.999
```

---

## Alerting Strategy

### Alert Pyramid

```
        Critical (Page)
       /                 \
      High (Notify)
     /                     \
    Medium (Ticket)
   /                         \
  Low (Log)
```

### Alert Rules

```yaml
# Prometheus alerts.yml
groups:
  - name: payment-service
    interval: 30s
    rules:
      # Critical - Page on-call
      - alert: PaymentServiceDown
        expr: up{job="payment-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Payment service is down"
          description: "{{ $labels.instance }} is unreachable"

      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5..",job="payment-service"}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{job="payment-service"}[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # High - Notify team
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_requests_seconds_bucket{job="payment-service"}[5m])) by (le)
          ) > 1
        for: 10m
        labels:
          severity: high
        annotations:
          summary: "High latency detected"
          description: "p95 latency is {{ $value }}s"

      # Medium - Create ticket
      - alert: HighMemoryUsage
        expr: |
          jvm_memory_used_bytes{job="payment-service"}
          /
          jvm_memory_max_bytes{job="payment-service"}
          > 0.8
        for: 15m
        labels:
          severity: medium
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"
```

---

## Production Best Practices

### ✅ DO

- Use structured logging (JSON)
- Include correlation IDs in all logs
- Set up distributed tracing
- Define SLIs/SLOs before deployment
- Monitor error budgets
- Use percentiles (p95, p99) not averages
- Tag metrics appropriately
- Implement health checks
- Set up alerting on SLO violations
- Create runbooks for alerts

### ❌ DON'T

- Log sensitive data (PII, passwords)
- Use high-cardinality dimensions in metrics
- Alert on symptoms, not causes
- Ignore alert fatigue
- Use only averages for latency
- Log at DEBUG level in production
- Create alerts without runbooks
- Monitor everything without priorities
- Ignore log volume costs

---

*This guide provides comprehensive patterns for implementing production-grade observability. Combine metrics, logs, and traces to gain full visibility into your distributed systems.*
