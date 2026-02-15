# Performance Testing Mastery

A comprehensive guide to performance testing Java applications with JMeter, Gatling, K6, and production-grade load testing patterns.

---

## Table of Contents

1. [Performance Testing Fundamentals](#performance-testing-fundamentals)
2. [Types of Performance Tests](#types-of-performance-tests)
3. [Key Metrics](#key-metrics)
4. [Apache JMeter](#apache-jmeter)
5. [Gatling](#gatling)
6. [K6 Load Testing](#k6-load-testing)
7. [Spring Boot Testing](#spring-boot-testing)
8. [Database Performance Testing](#database-performance-testing)
9. [Kafka Performance Testing](#kafka-performance-testing)
10. [Analyzing Results](#analyzing-results)
11. [Production Patterns](#production-patterns)

---

## Performance Testing Fundamentals

### Why Performance Testing Matters

```
Production Issues Prevented by Performance Testing:

1. Slow Response Times
   - Users abandon slow applications
   - Revenue impact

2. System Crashes
   - Memory leaks under load
   - Connection pool exhaustion

3. Scalability Issues
   - Can't handle peak traffic
   - Black Friday failures

4. Resource Wastage
   - Over-provisioned infrastructure
   - Unnecessary cloud costs

5. Poor User Experience
   - Timeouts and errors
   - Customer churn
```

### Performance Testing Principles

```
1. Test Early and Often
   - Performance testing in CI/CD
   - Catch regressions early

2. Test Realistic Scenarios
   - Production-like data volumes
   - Realistic user behavior

3. Measure Everything
   - Response time
   - Throughput
   - Resource utilization

4. Establish Baselines
   - Know your normal performance
   - Track trends over time

5. Test Under Various Conditions
   - Normal load
   - Peak load
   - Stress conditions
```

---

## Types of Performance Tests

### Load Testing

```
Goal: Verify system behavior under expected load

Example:
- 1000 concurrent users
- Normal traffic patterns
- Expected response times < 200ms
- No errors

When to Run:
- Before each release
- After infrastructure changes
- Continuously in staging
```

### Stress Testing

```
Goal: Find breaking point

Example:
- Gradually increase load
- 100 → 500 → 1000 → 2000 users
- Find where system fails
- Identify bottlenecks

When to Run:
- Capacity planning
- Before major events
- After architecture changes
```

### Spike Testing

```
Goal: Test sudden traffic increases

Example:
- Normal: 100 users
- Spike: 5000 users for 2 minutes
- Return: 100 users
- Verify recovery

When to Run:
- Flash sale preparation
- Product launches
- Marketing campaigns
```

### Soak Testing (Endurance Testing)

```
Goal: Find memory leaks and stability issues

Example:
- Moderate load (500 users)
- Extended duration (24-72 hours)
- Monitor memory, connections
- Check for degradation

When to Run:
- Before major releases
- After memory-related changes
- Quarterly baseline
```

---

## Key Metrics

### Response Time Metrics

```java
// Understanding Percentiles
Response Times for 1000 requests:

p50 (median): 150ms  - Half of requests faster
p90: 300ms           - 90% of requests faster
p95: 450ms           - 95% of requests faster
p99: 800ms           - 99% of requests faster
p99.9: 2000ms        - 99.9% of requests faster

Why p95/p99 Matter:
- Average hides outliers
- p95 shows user experience
- SLOs often use p95 or p99

Example SLO:
"95% of API requests complete in < 200ms"
```

### Throughput

```
Requests per Second (RPS)
- How many requests handled per second
- Example: 1000 RPS

Transactions per Second (TPS)
- How many business transactions per second
- Example: 500 orders/second

Target Setting:
Peak Traffic = Normal Traffic × 3-5×
Safety Margin = Peak Traffic × 2×

Example:
- Normal: 200 RPS
- Peak: 1000 RPS (5× normal)
- Test Target: 2000 RPS (2× peak)
```

### Resource Utilization

```
CPU Usage:
- Target: < 70% under peak load
- Leaves headroom for spikes
- Prevents CPU starvation

Memory:
- Heap usage: < 80% of max
- GC frequency: < 1/minute
- GC pause: < 100ms

Connections:
- Database pool: < 80% utilized
- HTTP connections: < 80% utilized

Disk I/O:
- Queue depth
- IOPS utilization
- Latency
```

---

## Apache JMeter

### Basic Test Plan

```xml
<!-- test-plan.jmx -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan">
      <stringProp name="TestPlan.comments">Payment API Load Test</stringProp>
      <elementProp name="TestPlan.user_defined_variables">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_URL">
            <stringProp name="Argument.value">http://localhost:8080</stringProp>
          </elementProp>
          <elementProp name="NUM_THREADS">
            <stringProp name="Argument.value">100</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>

    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup">
        <stringProp name="ThreadGroup.num_threads">${NUM_THREADS}</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>

        <HTTPSamplerProxy guiclass="HttpTestSampleGui">
          <stringProp name="HTTPSampler.domain">${BASE_URL}</stringProp>
          <stringProp name="HTTPSampler.path">/api/payments</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>

          <elementProp name="HTTPsampler.Arguments">
            <collectionProp name="Arguments.arguments">
              <elementProp name="">
                <stringProp name="Argument.value">
                  {
                    "amount": ${__Random(10,1000)},
                    "currency": "USD",
                    "userId": "user-${__threadNum}"
                  }
                </stringProp>
              </elementProp>
            </collectionProp>
          </elementProp>
        </HTTPSamplerProxy>
      </ThreadGroup>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### Running JMeter

```bash
# CLI mode (headless)
jmeter -n -t test-plan.jmx -l results.jtl -e -o reports/

# With properties
jmeter -n -t test-plan.jmx \
  -JNUM_THREADS=500 \
  -JBASE_URL=https://api.example.com \
  -l results.jtl

# Distributed testing
jmeter -n -t test-plan.jmx \
  -R server1:1099,server2:1099,server3:1099 \
  -l results.jtl
```

---

## Gatling

### Scala Simulation

```scala
package simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class PaymentLoadTest extends Simulation {

  // HTTP configuration
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling Performance Test")

  // Feeder for test data
  val paymentFeeder = Iterator.continually(Map(
    "amount" -> (scala.util.Random.nextInt(990) + 10),
    "currency" -> "USD",
    "userId" -> s"user-${scala.util.Random.nextInt(10000)}"
  ))

  // Scenario definition
  val createPayment = scenario("Create Payment")
    .feed(paymentFeeder)
    .exec(
      http("Create Payment")
        .post("/api/payments")
        .body(StringBody("""{
          "amount": ${amount},
          "currency": "${currency}",
          "userId": "${userId}"
        }""")).asJson
        .check(status.is(201))
        .check(jsonPath("$.id").saveAs("paymentId"))
    )
    .pause(1.second, 3.seconds)
    .exec(
      http("Get Payment")
        .get("/api/payments/${paymentId}")
        .check(status.is(200))
    )

  val getPayments = scenario("List Payments")
    .feed(paymentFeeder)
    .exec(
      http("List Payments")
        .get("/api/payments?userId=${userId}")
        .check(status.is(200))
        .check(jsonPath("$.content").exists)
    )

  // Load simulation
  setUp(
    createPayment.inject(
      rampUsers(100) during (30.seconds),
      constantUsersPerSec(50) during (5.minutes)
    ),
    getPayments.inject(
      rampUsers(50) during (30.seconds),
      constantUsersPerSec(100) during (5.minutes)
    )
  ).protocols(httpProtocol)
    .assertions(
      global.responseTime.percentile3.lt(300), // p99.9 < 300ms
      global.responseTime.percentile4.lt(500), // p99.99 < 500ms
      global.successfulRequests.percent.gt(99.9) // > 99.9% success
    )
}
```

### Advanced Gatling Patterns

```scala
// Spike Test
val spikeTest = scenario("Spike Test")
  .exec(/* scenario steps */)

setUp(
  spikeTest.inject(
    constantUsersPerSec(100) during (1.minute),  // Normal
    constantUsersPerSec(1000) during (2.minutes), // Spike
    constantUsersPerSec(100) during (1.minute)   // Recovery
  )
)

// Stress Test - Ramp up until failure
val stressTest = scenario("Stress Test")
  .exec(/* scenario steps */)

setUp(
  stressTest.inject(
    incrementUsersPerSec(50)
      .times(20)
      .eachLevelLasting(30.seconds)
      .separatedByRampsLasting(10.seconds)
      .startingFrom(100)
  )
)

// Closed workload model (connection pool simulation)
val closedModel = scenario("Closed Model")
  .exec(/* scenario steps */)

setUp(
  closedModel.inject(
    constantConcurrentUsers(100) during (5.minutes)
  )
)
```

### Running Gatling

```bash
# Run simulation
mvn gatling:test -Dgatling.simulationClass=simulations.PaymentLoadTest

# With environment variables
mvn gatling:test \
  -Dgatling.simulationClass=simulations.PaymentLoadTest \
  -DBASE_URL=https://api.production.com \
  -DUSERS=500

# CI/CD integration
mvn gatling:test \
  -Dgatling.simulationClass=simulations.PaymentLoadTest \
  -Dgatling.noReports=false \
  -Dgatling.resultsFolder=target/gatling
```

---

## K6 Load Testing

### JavaScript Test Script

```javascript
// payment-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const paymentDuration = new Trend('payment_duration');
const paymentCount = new Counter('payments_created');

// Test configuration
export const options = {
  stages: [
    { duration: '30s', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 users
    { duration: '30s', target: 500 },  // Spike to 500 users
    { duration: '2m', target: 500 },   // Stay at 500
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<300', 'p(99)<500'],
    'http_req_failed': ['rate<0.01'], // < 1% errors
    'errors': ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function() {
  // Create payment
  const payload = JSON.stringify({
    amount: Math.floor(Math.random() * 1000) + 10,
    currency: 'USD',
    userId: `user-${Math.floor(Math.random() * 10000)}`
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };

  const startTime = Date.now();

  const response = http.post(
    `${BASE_URL}/api/payments`,
    payload,
    params
  );

  const duration = Date.now() - startTime;
  paymentDuration.add(duration);

  // Validation
  const success = check(response, {
    'status is 201': (r) => r.status === 201,
    'has payment ID': (r) => r.json('id') !== undefined,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  if (success) {
    paymentCount.add(1);
  } else {
    errorRate.add(1);
    console.error(`Payment creation failed: ${response.status}`);
  }

  // Get payment
  if (response.status === 201) {
    const paymentId = response.json('id');
    const getResponse = http.get(
      `${BASE_URL}/api/payments/${paymentId}`,
      params
    );

    check(getResponse, {
      'get status is 200': (r) => r.status === 200,
    });
  }

  sleep(1); // Think time
}

// Setup function (runs once)
export function setup() {
  console.log('Starting performance test...');
  console.log(`Target: ${BASE_URL}`);

  // Health check
  const healthResponse = http.get(`${BASE_URL}/actuator/health`);
  if (healthResponse.status !== 200) {
    throw new Error('Application is not healthy');
  }

  return { startTime: Date.now() };
}

// Teardown function (runs once)
export function teardown(data) {
  const duration = (Date.now() - data.startTime) / 1000;
  console.log(`Test completed in ${duration}s`);
}
```

### Running K6

```bash
# Basic run
k6 run payment-load-test.js

# With environment variables
k6 run \
  -e BASE_URL=https://api.example.com \
  payment-load-test.js

# With VUs override
k6 run --vus 500 --duration 5m payment-load-test.js

# Cloud execution
k6 cloud payment-load-test.js

# Output results to InfluxDB
k6 run \
  --out influxdb=http://localhost:8086/k6 \
  payment-load-test.js
```

---

## Spring Boot Testing

### WebTestClient Load Test

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class PaymentApiPerformanceTest {

    @Autowired
    private WebTestClient webClient;

    @Test
    void loadTest_createPayments() throws Exception {
        int totalRequests = 10000;
        int concurrency = 100;

        ExecutorService executor = Executors.newFixedThreadPool(concurrency);
        CountDownLatch latch = new CountDownLatch(totalRequests);

        List<Long> responseTimes = new CopyOnWriteArrayList<>();
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger errorCount = new AtomicInteger(0);

        long startTime = System.currentTimeMillis();

        for (int i = 0; i < totalRequests; i++) {
            executor.submit(() -> {
                try {
                    long reqStart = System.nanoTime();

                    webClient.post()
                        .uri("/api/payments")
                        .bodyValue(createPaymentRequest())
                        .exchange()
                        .expectStatus().isCreated()
                        .expectBody()
                        .returnResult();

                    long reqEnd = System.nanoTime();
                    responseTimes.add((reqEnd - reqStart) / 1_000_000); // ms
                    successCount.incrementAndGet();

                } catch (Exception e) {
                    errorCount.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(5, TimeUnit.MINUTES);
        executor.shutdown();

        long endTime = System.currentTimeMillis();
        long totalDuration = endTime - startTime;

        // Calculate metrics
        PerformanceMetrics metrics = calculateMetrics(
            responseTimes,
            successCount.get(),
            errorCount.get(),
            totalDuration
        );

        // Assert SLOs
        assertThat(metrics.getP95()).isLessThan(300); // p95 < 300ms
        assertThat(metrics.getP99()).isLessThan(500); // p99 < 500ms
        assertThat(metrics.getSuccessRate()).isGreaterThan(99.0); // > 99%

        // Log results
        log.info("Performance Test Results:");
        log.info("Total Requests: {}", totalRequests);
        log.info("Successful: {}", successCount.get());
        log.info("Failed: {}", errorCount.get());
        log.info("Duration: {}ms", totalDuration);
        log.info("Throughput: {} req/s", metrics.getThroughput());
        log.info("p50: {}ms", metrics.getP50());
        log.info("p95: {}ms", metrics.getP95());
        log.info("p99: {}ms", metrics.getP99());
    }

    private PerformanceMetrics calculateMetrics(
            List<Long> responseTimes,
            int successCount,
            int errorCount,
            long totalDuration) {

        Collections.sort(responseTimes);

        return PerformanceMetrics.builder()
            .p50(getPercentile(responseTimes, 50))
            .p95(getPercentile(responseTimes, 95))
            .p99(getPercentile(responseTimes, 99))
            .successRate((double) successCount / (successCount + errorCount) * 100)
            .throughput((double) (successCount + errorCount) / totalDuration * 1000)
            .build();
    }

    private long getPercentile(List<Long> sorted, int percentile) {
        int index = (int) Math.ceil(percentile / 100.0 * sorted.size()) - 1;
        return sorted.get(Math.max(0, index));
    }
}
```

---

## Production Patterns

### CI/CD Integration

```yaml
# .github/workflows/performance-test.yml
name: Performance Tests

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  performance-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 21
        uses: actions/setup-java@v3
        with:
          java-version: '21'

      - name: Start application
        run: |
          docker-compose up -d
          ./wait-for-it.sh localhost:8080

      - name: Run Gatling tests
        run: mvn gatling:test

      - name: Check performance thresholds
        run: |
          python scripts/check-performance-thresholds.py \
            --results target/gatling/*/simulation.log \
            --p95-threshold 300 \
            --p99-threshold 500 \
            --error-rate-threshold 1.0

      - name: Upload Gatling report
        uses: actions/upload-artifact@v3
        with:
          name: gatling-report
          path: target/gatling/

      - name: Comment PR with results
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('target/gatling/summary.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.name,
              body: `## Performance Test Results\n\n\`\`\`\n${report}\n\`\`\``
            });
```

### Monitoring During Tests

```java
@Component
public class PerformanceTestMonitor {

    private final MeterRegistry meterRegistry;
    private final MemoryMXBean memoryMXBean;
    private final ThreadMXBean threadMXBean;

    @Scheduled(fixedRate = 5000)
    public void recordMetrics() {
        // Memory metrics
        MemoryUsage heapUsage = memoryMXBean.getHeapMemoryUsage();
        meterRegistry.gauge("jvm.memory.heap.used",
            heapUsage.getUsed());
        meterRegistry.gauge("jvm.memory.heap.committed",
            heapUsage.getCommitted());

        // Thread metrics
        meterRegistry.gauge("jvm.threads.live",
            threadMXBean.getThreadCount());
        meterRegistry.gauge("jvm.threads.daemon",
            threadMXBean.getDaemonThreadCount());

        // GC metrics
        long gcTime = getGCTime();
        meterRegistry.counter("jvm.gc.time").increment(gcTime);
    }

    private long getGCTime() {
        return ManagementFactory.getGarbageCollectorMXBeans().stream()
            .mapToLong(GarbageCollectorMXBean::getCollectionTime)
            .sum();
    }
}
```

---

## Best Practices

### ✅ DO

- Establish performance baselines
- Test with production-like data volumes
- Run tests in isolated environments
- Monitor system resources during tests
- Use percentiles (p95, p99) not averages
- Automate performance testing in CI/CD
- Test gradually increasing loads
- Simulate realistic user behavior
- Use correlation IDs for tracing
- Document performance requirements as SLOs

### ❌ DON'T

- Test against production
- Use only average response time
- Ignore resource utilization
- Skip warm-up periods
- Test with unrealistic data
- Run tests from developer machines
- Ignore failed requests in metrics
- Test only happy paths
- Skip spike and soak tests
- Forget to clean up test data

---

*This guide provides production-ready patterns for performance testing. Performance is a feature - test it continuously and measure everything that matters.*
