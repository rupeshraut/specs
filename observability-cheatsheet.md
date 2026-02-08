# 👁️ Observability Cheat Sheet

> **Purpose:** Production-grade observability — metrics, logging, tracing, alerting, and SLI/SLO definitions. Reference before instrumenting any service, defining alerts, or investigating production issues.
> **Stack context:** Java 21+ / Spring Boot 3.x / Micrometer / Prometheus / Grafana / OpenTelemetry / ELK/Loki

---

## 📋 The Three Pillars + Context

```
                    OBSERVABILITY
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐    ┌──────▼──────┐   ┌────▼────┐
   │ METRICS  │    │   LOGGING    │   │ TRACES  │
   │ (Numbers)│    │   (Events)   │   │ (Flows) │
   └────┬────┘    └──────┬──────┘   └────┬────┘
        │                │                │
   What is the      What happened     Where did the
   system DOING?    at this MOMENT?   request GO?
        │                │                │
   Prometheus/       ELK / Loki /     Jaeger / Tempo /
   Datadog/          CloudWatch       Zipkin / OTLP
   CloudWatch
        │                │                │
        └────────────────┼────────────────┘
                         │
                    ┌────▼────┐
                    │ ALERTS  │
                    │ (Action)│
                    └─────────┘
                  When should humans
                  be NOTIFIED?
```

### The Observability Decision: Which Pillar When?

| Question | Pillar | Tool |
|----------|--------|------|
| "Is the system healthy RIGHT NOW?" | **Metrics** | Dashboard gauges |
| "What's the p99 latency trend this week?" | **Metrics** | Time-series graph |
| "WHY did payment pay-123 fail?" | **Logs** | Log search by correlation ID |
| "WHERE did the request spend time?" | **Traces** | Distributed trace view |
| "Is the SLO being met this month?" | **Metrics** | SLO burn rate dashboard |
| "Should someone wake up at 3am?" | **Alerts** | Alert rule evaluation |

---

## 📊 Pattern 1: Metrics with Micrometer

### Metric Types — When to Use Each

```
COUNTER     — Things that only go UP
              "How many payments processed?"
              "How many errors occurred?"
              Use: rate() in Prometheus to see per-second throughput

GAUGE       — Current VALUE that goes up and down
              "How many connections are active?"
              "What's the current queue depth?"
              "How much heap memory is used?"

TIMER       — DURATION of things + count
              "How long do gateway calls take?"
              "What's the p99 request latency?"
              Combines: counter (of invocations) + distribution (of durations)

DISTRIBUTION — Distribution of VALUES (not time)
SUMMARY        "What's the payment amount distribution?"
               "How large are Kafka messages?"
```

### Custom Business Metrics

```java
@Component
public class PaymentObservability {

    private final MeterRegistry registry;

    // ── Counters: what happened ──
    public void recordPaymentOutcome(PaymentType type, String outcome, String gateway) {
        registry.counter("payment.outcome",
            "type", type.name(),
            "outcome", outcome,         // "success", "failed", "rejected", "timeout"
            "gateway", gateway
        ).increment();
    }

    public void recordRetryAttempt(String service, int attempt, boolean succeeded) {
        registry.counter("resilience.retry",
            "service", service,
            "attempt", String.valueOf(attempt),
            "succeeded", String.valueOf(succeeded)
        ).increment();
    }

    public void recordDeadLetter(String topic, String errorType) {
        registry.counter("kafka.dead_letter",
            "original_topic", topic,
            "error_type", errorType
        ).increment();
    }

    // ── Timers: how long things take ──
    public <T> T timeOperation(String name, String service, Supplier<T> operation) {
        return registry.timer("operation.duration",
            "name", name,
            "service", service
        ).record(operation);
    }

    // Convenient timer for gateway calls with outcome tagging
    public <T> T timeGatewayCall(String gateway, Supplier<T> call) {
        var sample = Timer.start(registry);
        String outcome = "success";
        try {
            return call.get();
        } catch (Exception e) {
            outcome = classifyError(e);
            throw e;
        } finally {
            sample.stop(registry.timer("gateway.call.duration",
                "gateway", gateway,
                "outcome", outcome));
        }
    }

    // ── Gauges: current state ──
    @PostConstruct
    public void registerGauges() {
        // Outbox backlog
        Gauge.builder("outbox.pending.count", outboxRepository,
                repo -> repo.countByStatus(OutboxStatus.PENDING))
            .description("Number of outbox events awaiting publish")
            .register(registry);

        // Circuit breaker states (0=closed, 1=half-open, 2=open)
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb ->
            Gauge.builder("circuit_breaker.state", cb,
                    b -> switch (b.getState()) {
                        case CLOSED -> 0; case HALF_OPEN -> 1; case OPEN -> 2;
                        default -> -1;
                    })
                .tag("name", cb.getName())
                .register(registry));

        // Active virtual threads (approximate)
        Gauge.builder("jvm.threads.virtual.active",
                () -> Thread.getAllStackTraces().keySet().stream()
                    .filter(Thread::isVirtual).count())
            .register(registry);
    }

    // ── Distribution summaries: value distributions ──
    public void recordPaymentAmount(BigDecimal amount, String currency) {
        registry.summary("payment.amount",
            "currency", currency
        ).record(amount.doubleValue());
    }

    public void recordKafkaMessageSize(String topic, int sizeBytes) {
        registry.summary("kafka.message.size",
            "topic", topic
        ).record(sizeBytes);
    }

    // ── SLI recording ──
    public void recordSli(String sliName, boolean good) {
        registry.counter("sli.events",
            "sli", sliName,
            "quality", good ? "good" : "bad"
        ).increment();
    }

    private String classifyError(Exception e) {
        if (e instanceof TimeoutException) return "timeout";
        if (e instanceof ConnectException) return "connection_refused";
        if (e instanceof HttpServerErrorException) return "server_error";
        if (e instanceof HttpClientErrorException) return "client_error";
        return "unknown";
    }
}
```

### Metric Naming Conventions

```
Format: <domain>.<entity>.<action>[.<detail>]

✅ Good names:
  payment.processed.total           (counter)
  payment.gateway.duration          (timer)
  payment.amount                    (distribution)
  kafka.consumer.lag                (gauge)
  kafka.dead_letter.total           (counter)
  outbox.pending.count              (gauge)
  circuit_breaker.state             (gauge)
  http.server.requests              (timer — auto by Spring)

❌ Bad names:
  paymentCount                      (no dots, no unit context)
  process_time                      (vague — what process?)
  errors                            (what kind? where?)
  data                              (meaningless)

Tag rules:
  • Low cardinality ONLY (< 100 unique values per tag)
  • ❌ NEVER tag with: userId, paymentId, IP address, email
  • ✅ Tag with: status, type, service, gateway, outcome, region
```

### Histogram Bucket Configuration

```yaml
management:
  metrics:
    distribution:
      # Enable percentile histograms for specific metrics
      percentiles-histogram:
        http.server.requests: true
        gateway.call.duration: true
        payment.gateway.duration: true
      # Define percentiles to publish
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99
        gateway.call.duration: 0.5, 0.95, 0.99
      # SLA boundaries (for counting requests within latency bands)
      slo:
        http.server.requests: 50ms, 100ms, 250ms, 500ms, 1s, 5s
      # Min/max expected values (optimizes bucket distribution)
      minimum-expected-value:
        http.server.requests: 1ms
        gateway.call.duration: 10ms
      maximum-expected-value:
        http.server.requests: 10s
        gateway.call.duration: 30s
```

---

## 📝 Pattern 2: Structured Logging

### Log Levels — When to Use Each

```
TRACE   Detailed debugging — variable values, loop iterations
        NEVER in production. Enabled temporarily for specific loggers.

DEBUG   Diagnostic info — method entry/exit, decision branches
        Disabled in prod by default. Enable per-logger for investigation.

INFO    Business events — payment processed, order created, service started
        The "story" of what the system is doing. Primary production level.

WARN    Unexpected but handled — retry triggered, fallback used, slow query
        System continues normally but something is suboptimal.

ERROR   Failure requiring attention — unhandled exception, DLT message,
        compensation failure. Should trigger an alert if sustained.

FATAL   System cannot continue — missing config, corrupt state
        (Rarely used in Spring Boot — usually throws and exits)
```

### Structured Logging Configuration

```xml
<!-- logback-spring.xml -->
<configuration>

    <!-- ── Dev profile: human-readable ── -->
    <springProfile name="local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <!-- ── Production: JSON for log aggregation ── -->
    <springProfile name="prod,staging">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <includeMdcKeyName>correlationId</includeMdcKeyName>
                <includeMdcKeyName>paymentId</includeMdcKeyName>
                <includeMdcKeyName>customerId</includeMdcKeyName>
                <customFields>
                    {"service":"${SERVICE_NAME:-payment-service}",
                     "environment":"${ENVIRONMENT:-unknown}",
                     "instance":"${HOSTNAME:-unknown}"}
                </customFields>
                <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                    <maxDepthPerThrowable>30</maxDepthPerThrowable>
                    <shortenedClassNameLength>36</shortenedClassNameLength>
                </throwableConverter>
            </encoder>
        </appender>
        <root level="WARN">
            <appender-ref ref="JSON" />
        </root>
        <logger name="com.example.payment" level="INFO" />
    </springProfile>

</configuration>
```

### MDC Context Propagation

```java
// ── Request context filter (HTTP) ──
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ObservabilityFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {

        String traceId = Optional.ofNullable(request.getHeader("X-Trace-Id"))
            .orElse(UUID.randomUUID().toString());
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
            .orElse(traceId);

        MDC.put("traceId", traceId);
        MDC.put("correlationId", correlationId);
        MDC.put("method", request.getMethod());
        MDC.put("path", request.getRequestURI());
        MDC.put("clientIp", request.getRemoteAddr());

        response.setHeader("X-Trace-Id", traceId);

        try {
            chain.doFilter(request, response);
        } finally {
            // Log request completion
            log.info("Request completed: {} {} → {} in {}ms",
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), System.currentTimeMillis() - getStartTime(request));
            MDC.clear();
        }
    }
}

// ── Kafka consumer context ──
@Component
public class KafkaObservabilityInterceptor implements RecordInterceptor<String, Object> {

    @Override
    public ConsumerRecord<String, Object> intercept(ConsumerRecord<String, Object> record,
            Consumer<String, Object> consumer) {
        MDC.put("kafkaTopic", record.topic());
        MDC.put("kafkaPartition", String.valueOf(record.partition()));
        MDC.put("kafkaOffset", String.valueOf(record.offset()));

        extractHeader(record, "correlationId").ifPresent(id -> MDC.put("correlationId", id));
        extractHeader(record, "traceId").ifPresent(id -> MDC.put("traceId", id));
        extractHeader(record, "eventId").ifPresent(id -> MDC.put("eventId", id));

        return record;
    }

    @Override
    public void afterRecord(ConsumerRecord<String, Object> record,
            Consumer<String, Object> consumer) {
        MDC.clear();
    }
}

// ── MDC propagation to virtual threads / CompletableFuture ──
@Component
public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            if (context != null) MDC.setContextMap(context);
            try {
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

### What to Log — Decision Guide

```
✅ ALWAYS LOG (INFO):
  • Service startup/shutdown with config summary
  • Business events (payment processed, order created)
  • External call entry/exit with timing
  • State transitions (payment INITIATED → CAPTURED)
  • Authentication events (login, token refresh)

✅ ALWAYS LOG (WARN):
  • Retry attempts with count and reason
  • Circuit breaker state changes
  • Fallback activations
  • Slow queries (> threshold)
  • Approaching resource limits (pool 80% full)

✅ ALWAYS LOG (ERROR):
  • Unhandled exceptions with full stack trace
  • Dead-letter messages with payload context
  • Compensation failures
  • Data inconsistencies detected
  • External service permanent failures

❌ NEVER LOG:
  • Passwords, tokens, API keys, secrets
  • Full credit card numbers (PCI)
  • PII in plain text (use masking)
  • Every SQL/MongoDB query (use DEBUG level)
  • Success of routine health checks
  • Request/response bodies in production (use DEBUG)
```

### PII Masking

```java
@Component
public class PiiMasker {

    public static String maskEmail(String email) {
        if (email == null) return null;
        int atIdx = email.indexOf('@');
        if (atIdx <= 1) return "***@" + email.substring(atIdx + 1);
        return email.charAt(0) + "***" + email.substring(atIdx);
    }

    public static String maskCardNumber(String number) {
        if (number == null || number.length() < 4) return "****";
        return "****-****-****-" + number.substring(number.length() - 4);
    }

    public static String maskAccountId(String id) {
        if (id == null || id.length() < 4) return "****";
        return "***" + id.substring(id.length() - 4);
    }
}

// Usage in logs
log.info("Processing payment for customer={}, card={}",
    PiiMasker.maskEmail(customer.email()),
    PiiMasker.maskCardNumber(card.number()));
// Output: "Processing payment for customer=j***@email.com, card=****-****-****-4242"
```

---

## 🔗 Pattern 3: Distributed Tracing

### OpenTelemetry Integration (Spring Boot 3.x)

```yaml
# application.yml — Spring Boot 3.x with Micrometer Tracing
management:
  tracing:
    sampling:
      probability: 1.0       # 100% in dev/staging, 0.1 (10%) in prod
    propagation:
      type: w3c              # W3C Trace Context standard
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_ENDPOINT:http://localhost:4318/v1/traces}
```

```xml
<!-- pom.xml dependencies -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Custom Spans for Business Logic

```java
@Component
public class PaymentService {

    private final ObservationRegistry observationRegistry;

    public PaymentResult processPayment(PaymentRequest request) {
        return Observation.createNotStarted("payment.process", observationRegistry)
            .lowCardinalityKeyValue("payment.type", request.type().name())
            .lowCardinalityKeyValue("payment.currency", request.currency())
            .highCardinalityKeyValue("payment.id", request.paymentId())    // For traces only
            .observe(() -> {
                validate(request);
                var fraudResult = checkFraud(request);
                var chargeResult = chargeGateway(request);
                return toResult(chargeResult);
            });
    }

    private FraudResult checkFraud(PaymentRequest request) {
        return Observation.createNotStarted("payment.fraud_check", observationRegistry)
            .lowCardinalityKeyValue("fraud.provider", "internal")
            .observe(() -> fraudService.evaluate(request));
    }

    private ChargeResult chargeGateway(PaymentRequest request) {
        return Observation.createNotStarted("payment.gateway_charge", observationRegistry)
            .lowCardinalityKeyValue("gateway", "stripe")
            .observe(() -> gateway.charge(request));
    }
}
```

### Trace Context Propagation Across Kafka

```java
// ── Producer: inject trace context into Kafka headers ──
@Component
public class TracingProducerInterceptor implements ProducerInterceptor<String, Object> {

    @Override
    public ProducerRecord<String, Object> onSend(ProducerRecord<String, Object> record) {
        // Micrometer Tracing auto-injects W3C traceparent header
        // when using ObservationRegistry-aware KafkaTemplate
        return record;
    }
}

// ── Spring Boot 3.x auto-propagation ──
// Just enable:
spring.kafka.producer.properties.interceptor.classes=\
  io.opentelemetry.instrumentation.kafkaclients.TracingProducerInterceptor
spring.kafka.consumer.properties.interceptor.classes=\
  io.opentelemetry.instrumentation.kafkaclients.TracingConsumerInterceptor
```

### Trace Visualization — What to See

```
Payment Request (trace: abc-123)
│
├── payment-service: POST /api/v1/payments (250ms total)
│   ├── payment.validate (5ms)
│   ├── payment.fraud_check (80ms)
│   │   └── fraud-service: POST /api/v1/evaluate (75ms)
│   │       └── mongodb: find (3ms)
│   ├── payment.gateway_charge (150ms)
│   │   └── stripe-api: POST /charges (145ms)   ← Bottleneck!
│   └── mongodb: insertOne (8ms)
│
├── kafka: payments.transaction.captured (async)
│   └── settlement-service: consume (45ms)
│       ├── mongodb: findAndModify (5ms)
│       └── notification-service: POST /notify (30ms)
│
└── kafka: payments.audit.log (async)
    └── audit-service: consume (10ms)
        └── elasticsearch: index (8ms)
```

---

## 🚨 Pattern 4: Alerting Strategy

### Alert Severity Levels

```
PAGE (Critical) — Wake someone up at 3am
  • Service completely down
  • Error rate > 5% for > 2 minutes
  • SLO burn rate critical
  • Data loss detected (DLT overflow)
  • Payment success rate dropped

TICKET (Warning) — Address during business hours
  • Elevated latency (p99 > 2x normal)
  • Consumer lag growing steadily
  • Retry rate > 20%
  • Circuit breaker flapping
  • Disk/memory approaching limit

INFO — Dashboard only, no notification
  • Deployment completed
  • Routine circuit breaker trip/recovery
  • Scheduled job completion
  • Traffic pattern changes
```

### Alert Rules — Production Templates

```yaml
groups:
  # ── Availability ──
  - name: availability
    rules:
      # Service down
      - alert: ServiceDown
        expr: up{job="payment-service"} == 0
        for: 1m
        labels:
          severity: page
        annotations:
          summary: "Payment service instance {{ $labels.instance }} is DOWN"

      # High error rate
      - alert: HighErrorRate
        expr: >
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))
          > 0.05
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "Error rate is {{ $value | humanizePercentage }} (> 5%)"

  # ── Latency ──
  - name: latency
    rules:
      # p99 latency spike
      - alert: HighP99Latency
        expr: >
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le))
          > 2
        for: 5m
        labels:
          severity: ticket
        annotations:
          summary: "p99 latency is {{ $value | humanizeDuration }} (> 2s)"

      # Gateway latency
      - alert: GatewayLatency
        expr: >
          histogram_quantile(0.95,
            sum(rate(gateway_call_duration_seconds_bucket[5m])) by (le, gateway))
          > 5
        for: 3m
        labels:
          severity: ticket
        annotations:
          summary: "Gateway {{ $labels.gateway }} p95 latency {{ $value | humanizeDuration }}"

  # ── Kafka ──
  - name: kafka
    rules:
      # Consumer lag
      - alert: KafkaConsumerLag
        expr: kafka_consumer_group_lag > 10000
        for: 10m
        labels:
          severity: ticket
        annotations:
          summary: "Consumer group {{ $labels.group }} lag: {{ $value }} on {{ $labels.topic }}"

      # Dead letters accumulating
      - alert: DeadLetterAccumulating
        expr: increase(kafka_dead_letter_total[1h]) > 50
        for: 0m
        labels:
          severity: page
        annotations:
          summary: "{{ $value }} dead letters in last hour on {{ $labels.original_topic }}"

  # ── Resources ──
  - name: resources
    rules:
      # Heap approaching limit
      - alert: HighHeapUsage
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        labels:
          severity: ticket

      # Connection pool exhaustion
      - alert: ConnectionPoolExhausted
        expr: mongodb_driver_pool_waitqueuesize > 0
        for: 30s
        labels:
          severity: ticket

  # ── Circuit Breakers ──
  - name: resilience
    rules:
      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state == 2
        for: 0m
        labels:
          severity: page
        annotations:
          summary: "Circuit breaker {{ $labels.name }} is OPEN"

      - alert: HighRetryRate
        expr: >
          sum(rate(resilience_retry_total{succeeded="false"}[5m])) by (service)
          /
          sum(rate(resilience_retry_total[5m])) by (service)
          > 0.3
        for: 5m
        labels:
          severity: ticket
```

### Alert Anti-Patterns

```
❌ Alerting on causes instead of symptoms
   "CPU > 80%" → what's the user impact?
   ✅ Better: "p99 latency > 2s" or "error rate > 5%"

❌ Too many non-actionable alerts → alert fatigue
   ✅ Every alert must have a documented runbook action

❌ No severity levels → everything pages at 3am
   ✅ page = customer impact, ticket = degrade gracefully

❌ Threshold too tight → flapping alerts
   ✅ Use 'for' duration to avoid transient spikes

❌ Missing alerts → find out from customers
   ✅ Alert on SLO burn rate — catches issues before customers notice
```

---

## 🎯 Pattern 5: SLI/SLO Definitions

### SLI Taxonomy for Payment Services

```
AVAILABILITY SLI:
  Definition: Proportion of successful requests
  Formula:    successful requests / total requests
  Good event: HTTP 2xx or 4xx (client error is "available" from server perspective)
  Bad event:  HTTP 5xx or timeout

LATENCY SLI:
  Definition: Proportion of requests faster than threshold
  Formula:    requests < 500ms / total requests
  Good event: Response time < 500ms
  Bad event:  Response time >= 500ms

CORRECTNESS SLI:
  Definition: Proportion of payments processed correctly
  Formula:    (payments - reconciliation mismatches) / payments
  Good event: Payment matches reconciliation
  Bad event:  Reconciliation mismatch

FRESHNESS SLI (for async):
  Definition: Proportion of events processed within SLA
  Formula:    events processed < 5min / total events
  Good event: Kafka consumer lag < 5 minutes
  Bad event:  Kafka consumer lag >= 5 minutes
```

### SLO Definitions

```yaml
# SLO Document — Payment Service
slos:
  availability:
    sli: "Successful HTTP responses (non-5xx) / total responses"
    target: 99.95%                    # ~22 min downtime/month
    window: 30 days (rolling)
    error_budget: 0.05%               # ~22 min/month
    burn_rate_alert:
      fast: 14.4x over 1h            # Consuming budget in ~2 days
      slow: 6x over 6h               # Consuming budget in ~5 days

  latency:
    sli: "Responses < 500ms / total responses"
    target: 99.0%                     # 1% of requests can be slow
    window: 30 days (rolling)
    tiers:
      p50: < 100ms
      p95: < 300ms
      p99: < 500ms

  payment_success:
    sli: "Successful payment captures / total payment attempts"
    target: 99.9%
    window: 7 days (rolling)
    exclusions:
      - "Legitimate fraud rejections"
      - "Invalid payment methods (client error)"

  event_freshness:
    sli: "Events processed within 5 minutes / total events"
    target: 99.5%
    window: 30 days (rolling)
```

### SLO Burn Rate Alerts (Multi-Window)

```yaml
# Fast burn — consuming error budget quickly (page immediately)
- alert: SLOFastBurn_Availability
  expr: >
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h]))
      /
      sum(rate(http_server_requests_seconds_count[1h]))
    ) > (14.4 * 0.0005)
    AND
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
      /
      sum(rate(http_server_requests_seconds_count[5m]))
    ) > (14.4 * 0.0005)
  for: 2m
  labels:
    severity: page
  annotations:
    summary: "Availability SLO fast burn: {{ $value | humanizePercentage }} error rate"

# Slow burn — steady error budget consumption (ticket)
- alert: SLOSlowBurn_Availability
  expr: >
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[6h]))
      /
      sum(rate(http_server_requests_seconds_count[6h]))
    ) > (6 * 0.0005)
    AND
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[30m]))
      /
      sum(rate(http_server_requests_seconds_count[30m]))
    ) > (6 * 0.0005)
  for: 15m
  labels:
    severity: ticket
  annotations:
    summary: "Availability SLO slow burn — error budget being consumed steadily"
```

---

## 📊 Pattern 6: Dashboards — What to Show

### Service Health Dashboard (The "Golden Signals")

```
Row 1: THE BIG NUMBERS
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ Request Rate │  Error Rate  │ p50 Latency  │ p99 Latency  │
│   1,234/sec  │    0.12%     │    45ms      │   320ms      │
└──────────────┴──────────────┴──────────────┴──────────────┘

Row 2: SLO STATUS
┌──────────────────────────────────────────────────────────┐
│ Availability: 99.97% (budget: 68% remaining)             │
│ Latency:      99.3%  (budget: 30% remaining) ⚠️          │
│ Payment Success: 99.95% (budget: 50% remaining)          │
└──────────────────────────────────────────────────────────┘

Row 3: TRAFFIC
┌──────────────────────────────────────────────────────────┐
│ [Request rate over time - line chart]                     │
│ [Error rate over time - line chart with threshold line]   │
│ [Latency percentiles over time - p50, p95, p99]          │
└──────────────────────────────────────────────────────────┘

Row 4: DEPENDENCIES
┌──────────────────────────────────────────────────────────┐
│ Payment Gateway: ✅ CLOSED (0.5% failure)                │
│ Fraud Service:   ✅ CLOSED (0.1% failure)                │
│ Account Service: ⚠️ HALF-OPEN (45% failure)              │
│ MongoDB:         ✅ Pool 23/50 (p99: 8ms)                │
│ Kafka:           ✅ Lag: 45 msgs (consumer: 3/3 running) │
└──────────────────────────────────────────────────────────┘

Row 5: INFRASTRUCTURE
┌──────────────────────────────────────────────────────────┐
│ [Heap usage over time]  [GC pause histogram]             │
│ [CPU usage]             [Thread count]                    │
│ [Connection pools]      [Kafka consumer lag]              │
└──────────────────────────────────────────────────────────┘
```

### Key Grafana Queries (PromQL)

```promql
# Request rate
sum(rate(http_server_requests_seconds_count[5m]))

# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m]))

# p99 latency
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))

# Availability (over 30 days)
1 - (
  sum(increase(http_server_requests_seconds_count{status=~"5.."}[30d]))
  / sum(increase(http_server_requests_seconds_count[30d]))
)

# Error budget remaining
1 - (
  sum(increase(http_server_requests_seconds_count{status=~"5.."}[30d]))
  / sum(increase(http_server_requests_seconds_count[30d]))
  / (1 - 0.9995)  # 99.95% SLO target
)

# Gateway success rate per gateway
sum(rate(gateway_call_duration_seconds_count{outcome="success"}[5m])) by (gateway)
/ sum(rate(gateway_call_duration_seconds_count[5m])) by (gateway)

# Kafka end-to-end latency (event time to processing time)
histogram_quantile(0.99, sum(rate(kafka_consumer_processing_lag_seconds_bucket[5m])) by (le, topic))
```

---

## 🚫 Observability Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **High-cardinality metric tags** | Prometheus memory explosion (userId, orderId) | Tags < 100 unique values |
| **Logging request/response bodies in prod** | Performance hit, PII exposure, disk fill | DEBUG level only, mask PII |
| **No correlation ID** | Can't trace a request across services | Propagate correlationId in every hop |
| **Metrics without alerts** | Dashboard exists but nobody watches it | Every metric that matters gets an alert |
| **Alerts without runbooks** | Alert fires, engineer doesn't know what to do | Document action for every alert |
| **Sampling traces at 100% in prod** | Storage cost, performance overhead | 10% sampling + always sample errors |
| **Alerting on causes, not symptoms** | "CPU high" doesn't mean users are affected | Alert on latency, errors, throughput |
| **No MDC cleanup** | Context leaks between requests on thread reuse | Always MDC.clear() in finally block |
| **Logging secrets** | Security breach in log aggregation | Automated secret scanning, PII masking |
| **Missing async context propagation** | Traces break at Kafka/async boundaries | Propagate trace context in headers |

---

## 💡 Golden Rules of Observability

```
1.  METRICS tell you WHAT is wrong. LOGS tell you WHY. TRACES tell you WHERE.
2.  Alert on SYMPTOMS (latency, errors) not CAUSES (CPU, memory).
3.  Every alert must have a RUNBOOK — if there's no action, it's not an alert.
4.  CORRELATION ID in every log line, every event, every trace — non-negotiable.
5.  SLOs define what "good" means — without them, you're guessing.
6.  Error budget is your friend — it tells you when to push features vs fix reliability.
7.  LOW CARDINALITY tags only — userId in a metric tag will bankrupt your monitoring.
8.  STRUCTURED JSON logging in production — grep is not an observability strategy.
9.  SAMPLE traces in production (10%) but ALWAYS capture errors and slow requests.
10. Observability is a PRODUCT FEATURE — invest in it before the first outage, not after.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Micrometer / Prometheus / Grafana / OpenTelemetry*
