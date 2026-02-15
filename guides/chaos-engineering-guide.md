# Chaos Engineering for Java Applications

A comprehensive guide to implementing chaos engineering practices for Java microservices with practical tools, patterns, and production-ready implementations.

---

## Table of Contents

1. [Chaos Engineering Fundamentals](#chaos-engineering-fundamentals)
2. [Principles of Chaos](#principles-of-chaos)
3. [Chaos Toolkit](#chaos-toolkit)
4. [Chaos Monkey for Spring Boot](#chaos-monkey-for-spring-boot)
5. [Litmus Chaos](#litmus-chaos)
6. [Simulating Failures](#simulating-failures)
7. [Testing Resilience Patterns](#testing-resilience-patterns)
8. [Observability During Chaos](#observability-during-chaos)
9. [Game Days](#game-days)
10. [Production Chaos](#production-chaos)

---

## Chaos Engineering Fundamentals

### What is Chaos Engineering?

```
Chaos Engineering is the discipline of experimenting on a system
to build confidence in the system's capability to withstand
turbulent conditions in production.

Key Principles:
1. Build a hypothesis around steady-state behavior
2. Vary real-world events
3. Run experiments in production
4. Automate experiments to run continuously
5. Minimize blast radius
```

### Why Chaos Engineering?

```
Traditional Testing vs Chaos Engineering:

Traditional Testing:
✓ Known failure modes
✓ Controlled environment
✓ Specific scenarios
✗ Can't predict all failures
✗ Production is different

Chaos Engineering:
✓ Unknown unknowns
✓ Production-like conditions
✓ Real system behavior
✓ Continuous validation
✓ Builds confidence

Real-World Benefits:
- Discover weaknesses before customers do
- Validate resilience patterns actually work
- Build confidence in system reliability
- Improve incident response
- Reduce MTTR (Mean Time To Recovery)
```

### Common Chaos Experiments

```
Infrastructure Chaos:
- Terminate instances
- Network latency/partition
- DNS failures
- Clock skew
- Disk full

Application Chaos:
- Kill processes
- Throw exceptions
- Slow down responses
- Return errors
- Fill memory

Dependency Chaos:
- Database failures
- Cache unavailable
- Third-party API errors
- Message queue failures
- Service mesh failures
```

---

## Principles of Chaos

### Scientific Method

```
1. Define Steady State
   - What does "normal" look like?
   - Metrics: throughput, latency, error rate
   - Example: 99% success rate, p95 < 200ms

2. Hypothesis
   - "We believe that terminating a pod will not
      affect user experience because we have 3 replicas"

3. Introduce Chaos
   - Terminate 1 of 3 pods
   - Measure impact

4. Analyze Results
   - Did steady state hold?
   - What broke?
   - Why?

5. Improve System
   - Fix discovered issues
   - Repeat experiment
```

### Start Small, Scale Gradually

```
Phase 1: Development Environment
- Safe experimentation
- Learn tools
- Develop playbooks

Phase 2: Staging/Pre-Production
- Production-like environment
- Larger blast radius
- Real dependencies

Phase 3: Production (Controlled)
- Limited scope (single region/zone)
- Off-peak hours
- Manual execution
- Team on standby

Phase 4: Production (Automated)
- Broader scope
- Business hours
- Automated experiments
- Continuous chaos
```

---

## Chaos Monkey for Spring Boot

### Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>3.1.0</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
chaos:
  monkey:
    enabled: true
    watcher:
      controller: true
      restController: true
      service: true
      repository: true
      component: false

management:
  endpoint:
    chaosmonkey:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,chaosmonkey

spring:
  profiles:
    active: chaos # Only enable in specific environments
```

### Assault Configuration

```yaml
# Latency Assault
chaos:
  monkey:
    assaults:
      latencyActive: true
      latencyRangeStart: 1000
      latencyRangeEnd: 5000
      level: 5  # 5% of requests

# Exception Assault
chaos:
  monkey:
    assaults:
      exceptionsActive: true
      exception:
        type: java.lang.RuntimeException
        arguments:
          - value: "Chaos Monkey - Simulated Exception"
      level: 3  # 3% of requests

# App Killer Assault
chaos:
  monkey:
    assaults:
      killApplicationActive: true
      killApplicationCronExpression: "*/30 * * * * ?" # Every 30 seconds

# Memory Assault
chaos:
  monkey:
    assaults:
      memoryActive: true
      memoryMillisecondsHoldFilledMemory: 10000
      memoryMillisecondsWaitNextIncrease: 1000
      memoryFillIncrementFraction: 0.15
      memoryFillTargetFraction: 0.25
```

### Runtime Control via Actuator

```bash
# Enable Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/enable

# Disable Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/disable

# Get current configuration
curl http://localhost:8080/actuator/chaosmonkey

# Enable latency assault
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H 'Content-Type: application/json' \
  -d '{
    "latencyActive": true,
    "latencyRangeStart": 1000,
    "latencyRangeEnd": 3000,
    "level": 10
  }'

# Get current status
curl http://localhost:8080/actuator/chaosmonkey/status

# Get watcher configuration
curl http://localhost:8080/actuator/chaosmonkey/watchers
```

### Custom Chaos Annotations

```java
@Service
public class PaymentService {

    // Disable chaos for critical operations
    @ChaosMonkeyIgnore
    public void processRefund(PaymentId id) {
        // Critical operation - never inject chaos
    }

    // Custom latency configuration
    @ChaosMonkeyLatency(min = 100, max = 500)
    public Payment getPayment(PaymentId id) {
        return paymentRepository.findById(id)
            .orElseThrow();
    }
}
```

---

## Simulating Failures

### Network Latency

```java
@Component
@ConditionalOnProperty("chaos.network.enabled")
public class NetworkChaosFilter implements Filter {

    @Value("${chaos.network.latency.min:100}")
    private int minLatency;

    @Value("${chaos.network.latency.max:1000}")
    private int maxLatency;

    @Value("${chaos.network.latency.probability:0.1}")
    private double probability;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        if (shouldInjectLatency()) {
            int delay = ThreadLocalRandom.current()
                .nextInt(minLatency, maxLatency);

            try {
                Thread.sleep(delay);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        chain.doFilter(request, response);
    }

    private boolean shouldInjectLatency() {
        return ThreadLocalRandom.current().nextDouble() < probability;
    }
}
```

### Database Failures

```java
@Aspect
@Component
@ConditionalOnProperty("chaos.database.enabled")
public class DatabaseChaosAspect {

    @Value("${chaos.database.error.probability:0.05}")
    private double errorProbability;

    @Around("@annotation(org.springframework.data.repository.Repository)")
    public Object injectDatabaseChaos(ProceedingJoinPoint joinPoint)
            throws Throwable {

        if (shouldInjectError()) {
            throw new DataAccessException("Chaos: Simulated database failure") {};
        }

        // Simulate slow query
        if (shouldInjectSlowQuery()) {
            Thread.sleep(ThreadLocalRandom.current().nextInt(2000, 5000));
        }

        return joinPoint.proceed();
    }

    private boolean shouldInjectError() {
        return ThreadLocalRandom.current().nextDouble() < errorProbability;
    }

    private boolean shouldInjectSlowQuery() {
        return ThreadLocalRandom.current().nextDouble() < errorProbability;
    }
}
```

### External Service Failures

```java
@Component
@ConditionalOnProperty("chaos.external.enabled")
public class ExternalServiceChaosInterceptor implements ClientHttpRequestInterceptor {

    @Value("${chaos.external.timeout.probability:0.1}")
    private double timeoutProbability;

    @Value("${chaos.external.error.probability:0.05}")
    private double errorProbability;

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                       ClientHttpRequestExecution execution)
            throws IOException {

        // Simulate timeout
        if (shouldSimulateTimeout()) {
            try {
                Thread.sleep(30000); // 30 second delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        // Simulate error response
        if (shouldSimulateError()) {
            return createErrorResponse();
        }

        return execution.execute(request, body);
    }

    private boolean shouldSimulateTimeout() {
        return ThreadLocalRandom.current().nextDouble() < timeoutProbability;
    }

    private boolean shouldSimulateError() {
        return ThreadLocalRandom.current().nextDouble() < errorProbability;
    }

    private ClientHttpResponse createErrorResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpStatusCode getStatusCode() {
                return HttpStatus.SERVICE_UNAVAILABLE;
            }

            @Override
            public String getStatusText() {
                return "Chaos: Simulated Service Unavailable";
            }

            @Override
            public void close() {}

            @Override
            public InputStream getBody() {
                return new ByteArrayInputStream("Service Unavailable".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                return new HttpHeaders();
            }
        };
    }
}
```

---

## Testing Resilience Patterns

### Circuit Breaker Validation

```java
@SpringBootTest
@TestPropertySource(properties = {
    "chaos.external.enabled=true",
    "chaos.external.error.probability=0.5"
})
class CircuitBreakerChaosTest {

    @Autowired
    private PaymentGatewayService paymentGateway;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Test
    void circuitBreaker_opensAfterFailures() throws Exception {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("paymentGateway");

        // Initial state
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.CLOSED);

        // Trigger failures (chaos will inject errors)
        int failures = 0;
        for (int i = 0; i < 20; i++) {
            try {
                paymentGateway.processPayment(createPayment());
            } catch (Exception e) {
                failures++;
            }
        }

        // Circuit breaker should open
        await().atMost(Duration.ofSeconds(5))
            .until(() -> circuitBreaker.getState() == CircuitBreaker.State.OPEN);

        // Verify fallback is used
        PaymentResult result = paymentGateway.processPayment(createPayment());
        assertThat(result.isFallback()).isTrue();

        log.info("Circuit breaker opened after {} failures", failures);
    }
}
```

### Retry Validation

```java
@SpringBootTest
@TestPropertySource(properties = {
    "chaos.external.enabled=true",
    "chaos.external.error.probability=0.3"
})
class RetryPolicyChaosTest {

    @Autowired
    private InventoryService inventoryService;

    @Test
    void retry_succeedsAfterTransientFailures() {
        AtomicInteger attempts = new AtomicInteger(0);

        // Mock to count attempts
        RetryListener listener = new RetryListener() {
            @Override
            public <T, E extends Throwable> void onRetry(RetryContext context) {
                attempts.incrementAndGet();
            }
        };

        RetryTemplate retryTemplate = inventoryService.getRetryTemplate();
        retryTemplate.registerListener(listener);

        // Execute with chaos - should eventually succeed
        StockLevel stock = inventoryService.checkStock("item-123");

        assertThat(stock).isNotNull();
        assertThat(attempts.get()).isGreaterThan(0);

        log.info("Operation succeeded after {} attempts", attempts.get());
    }
}
```

### Bulkhead Validation

```java
@SpringBootTest
@TestPropertySource(properties = {
    "chaos.external.enabled=true",
    "chaos.external.timeout.probability=0.5"
})
class BulkheadChaosTest {

    @Autowired
    private ReportingService reportingService;

    @Autowired
    private BulkheadRegistry bulkheadRegistry;

    @Test
    void bulkhead_protectsSystemUnderChaos() throws Exception {
        Bulkhead bulkhead = bulkheadRegistry.bulkhead("reporting");

        ExecutorService executor = Executors.newFixedThreadPool(50);
        CountDownLatch latch = new CountDownLatch(50);

        AtomicInteger rejected = new AtomicInteger(0);
        AtomicInteger completed = new AtomicInteger(0);

        // Flood with requests
        for (int i = 0; i < 50; i++) {
            executor.submit(() -> {
                try {
                    reportingService.generateReport();
                    completed.incrementAndGet();
                } catch (BulkheadFullException e) {
                    rejected.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(30, TimeUnit.SECONDS);

        // Verify bulkhead protected system
        assertThat(rejected.get()).isGreaterThan(0);
        assertThat(bulkhead.getMetrics().getAvailableConcurrentCalls())
            .isGreaterThanOrEqualTo(0);

        log.info("Bulkhead stats - Completed: {}, Rejected: {}",
            completed.get(), rejected.get());
    }
}
```

---

## Chaos Toolkit

### Installation

```bash
# Install Chaos Toolkit
pip install chaostoolkit
pip install chaostoolkit-spring

# Verify installation
chaos --version
```

### Experiment Definition

```json
{
  "version": "1.0.0",
  "title": "Payment Service Should Remain Available During Pod Termination",
  "description": "Verify that terminating one payment service pod does not affect availability",
  "tags": ["kubernetes", "availability"],

  "steady-state-hypothesis": {
    "title": "Services are healthy and responsive",
    "probes": [
      {
        "type": "probe",
        "name": "payment-service-is-healthy",
        "tolerance": true,
        "provider": {
          "type": "http",
          "url": "http://payment-service:8080/actuator/health",
          "timeout": 3
        }
      },
      {
        "type": "probe",
        "name": "payment-success-rate-above-99",
        "tolerance": {
          "type": "range",
          "range": [99.0, 100.0]
        },
        "provider": {
          "type": "python",
          "module": "chaosspring.probes",
          "func": "get_success_rate",
          "arguments": {
            "url": "http://prometheus:9090",
            "query": "rate(http_server_requests_total{status='200'}[1m]) * 100"
          }
        }
      }
    ]
  },

  "method": [
    {
      "type": "action",
      "name": "terminate-payment-pod",
      "provider": {
        "type": "python",
        "module": "chaosk8s.pod.actions",
        "func": "terminate_pods",
        "arguments": {
          "label_selector": "app=payment-service",
          "qty": 1,
          "ns": "production",
          "rand": true
        }
      },
      "pauses": {
        "after": 10
      }
    }
  ],

  "rollbacks": [
    {
      "type": "action",
      "name": "verify-pods-recovered",
      "provider": {
        "type": "python",
        "module": "chaosk8s.pod.probes",
        "func": "pods_in_phase",
        "arguments": {
          "label_selector": "app=payment-service",
          "phase": "Running",
          "ns": "production"
        }
      }
    }
  ]
}
```

### Running Experiments

```bash
# Validate experiment
chaos validate experiment.json

# Run experiment
chaos run experiment.json

# Run with report
chaos run experiment.json --journal-path=journal.json

# Run in verify mode (no chaos, just probes)
chaos run experiment.json --verify-only
```

---

## Litmus Chaos (Kubernetes)

### Installation

```bash
# Install Litmus
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v2.14.0.yaml

# Verify installation
kubectl get pods -n litmus

# Install chaos experiments
kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.14.0?file=charts/generic/experiments.yaml
```

### Pod Delete Experiment

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=payment-service
    appkind: deployment

  engineState: active
  chaosServiceAccount: litmus-admin

  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"

            - name: CHAOS_INTERVAL
              value: "10"

            - name: FORCE
              value: "false"

            - name: PODS_AFFECTED_PERC
              value: "50"
```

### Network Latency Experiment

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=order-service
    appkind: deployment

  engineState: active
  chaosServiceAccount: litmus-admin

  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_LATENCY
              value: "2000" # 2 second latency

            - name: TOTAL_CHAOS_DURATION
              value: "120"

            - name: TARGET_CONTAINER
              value: "order-service"

            - name: PODS_AFFECTED_PERC
              value: "100"

            - name: NETWORK_INTERFACE
              value: "eth0"

            - name: CONTAINER_RUNTIME
              value: "containerd"
```

---

## Game Days

### Planning a Game Day

```
1. Preparation (1-2 weeks before)
   - Define objectives
   - Select experiments
   - Notify stakeholders
   - Prepare runbooks
   - Set up monitoring dashboards

2. Team Assembly
   - Chaos coordinator
   - Application teams
   - SRE/DevOps
   - Product owners
   - Communications

3. Game Day Agenda
   09:00 - Kickoff & objectives
   09:30 - Baseline metrics review
   10:00 - Experiment 1: Pod termination
   10:30 - Analysis & discussion
   11:00 - Experiment 2: Database latency
   11:30 - Analysis & discussion
   12:00 - Lunch
   13:00 - Experiment 3: Network partition
   13:30 - Analysis & discussion
   14:00 - Experiment 4: Resource exhaustion
   14:30 - Analysis & discussion
   15:00 - Wrap-up & action items

4. Post-Game Day
   - Document findings
   - Create improvement tickets
   - Update runbooks
   - Schedule follow-up
```

### Example Game Day Scenarios

```yaml
# Scenario 1: Database Primary Failover
Objective: Verify application handles database failover gracefully
Expected Outcome: No user-visible errors, automatic reconnection
Blast Radius: Single service
Duration: 15 minutes

# Scenario 2: Kafka Broker Restart
Objective: Verify message processing resilience
Expected Outcome: Messages not lost, consumers reconnect
Blast Radius: Message processing
Duration: 20 minutes

# Scenario 3: Redis Cache Failure
Objective: Verify cache-aside pattern works
Expected Outcome: Slower responses, no errors
Blast Radius: Read operations
Duration: 10 minutes

# Scenario 4: Region Failure
Objective: Verify multi-region failover
Expected Outcome: Traffic shifts to backup region
Blast Radius: Entire region
Duration: 30 minutes
```

---

## Production Chaos

### Safety Guidelines

```
1. Minimize Blast Radius
   ✓ Start with single instance
   ✓ Single service
   ✓ Single region
   ✗ All instances
   ✗ Multiple services
   ✗ All regions

2. Have Kill Switch
   - Ability to stop chaos immediately
   - Automated rollback
   - Manual override

3. Monitor Continuously
   - Real-time dashboards
   - Alert thresholds
   - User impact metrics

4. Scheduled Windows
   - Off-peak hours initially
   - Business hours (advanced)
   - Avoid: holidays, launches, incidents

5. Communication
   - Notify stakeholders
   - Status page updates
   - Incident channels ready
```

### Production Chaos Checklist

```
Pre-Execution:
☐ Hypothesis documented
☐ Steady-state metrics defined
☐ Blast radius limited
☐ Team on standby
☐ Monitoring dashboards ready
☐ Rollback plan tested
☐ Stakeholders notified

During Execution:
☐ Monitor user impact
☐ Track system metrics
☐ Document observations
☐ Be ready to abort

Post-Execution:
☐ Analyze results
☐ Document learnings
☐ Create improvement tickets
☐ Update runbooks
☐ Share findings
```

---

## Best Practices

### ✅ DO

- Start with staging environments
- Minimize blast radius
- Define clear steady-state metrics
- Automate chaos experiments
- Run experiments regularly
- Document all findings
- Have rollback procedures
- Monitor user impact
- Communicate with stakeholders
- Build confidence gradually

### ❌ DON'T

- Start chaos in production
- Run experiments during incidents
- Inject chaos without hypothesis
- Ignore user impact
- Run chaos during peak hours (initially)
- Skip post-mortem analysis
- Forget to notify teams
- Exceed blast radius limits
- Run experiments back-to-back
- Assume resilience without testing

---

*This guide provides production-ready patterns for chaos engineering. Build confidence in your system by breaking it in controlled ways - because the only difference between chaos engineering and causing an outage is a hypothesis.*
