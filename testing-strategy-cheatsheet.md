# 🧪 Testing Strategy & Patterns Cheat Sheet

> **Purpose:** Production-grade testing patterns for enterprise distributed systems. Reference before writing any test class, designing a test harness, or setting up CI test stages.
> **Stack context:** Java 21+ / Spring Boot 3.x / JUnit 5 / Testcontainers / Kafka / MongoDB / ActiveMQ Artemis / Resilience4j

---

## 📋 The Testing Decision Framework

Before writing any test, answer:

| Question | Determines |
|----------|-----------|
| **What** am I testing? (unit of behavior, integration, contract) | Test type |
| **Why** could this break? (logic error, config, dependency, concurrency) | What to assert |
| **How fast** must this test run? | Unit vs integration boundary |
| **Who** will read this test when it fails? (me, teammate, CI bot) | Naming and clarity |
| **What's the blast radius** if this code has a bug? | Coverage depth |
| **Is this deterministic?** | Flakiness risk |

---

## 🏗️ The Test Pyramid — Adapted for Distributed Systems

```
                        ╱╲
                       ╱  ╲
                      ╱ E2E╲           Few: Critical user journeys only
                     ╱──────╲          Slow, expensive, fragile
                    ╱        ╲
                   ╱ Contract ╲        Moderate: API & event schema compatibility
                  ╱────────────╲       Fast, high value for microservices
                 ╱              ╲
                ╱  Integration   ╲     Moderate: Real DB, Kafka, queues
               ╱──────────────────╲    Testcontainers, seconds per test
              ╱                    ╲
             ╱    Component Tests   ╲  Moderate: Full Spring context, mocked externals
            ╱────────────────────────╲ Slower than unit, faster than integration
           ╱                          ╲
          ╱        Unit Tests          ╲  Many: Fast, isolated, deterministic
         ╱──────────────────────────────╲ Milliseconds per test, bulk of your suite
        ╱                                ╲
       ╱      Static Analysis / Linting   ╲ Automated: Runs on every commit
      ╱────────────────────────────────────╲ ArchUnit, SpotBugs, Checkstyle
```

### Coverage Targets by Layer

| Layer | Target Coverage | Run Frequency | Execution Time |
|-------|----------------|---------------|----------------|
| **Unit** | > 85% line, > 70% branch | Every build | < 30 seconds total |
| **Component** | Critical paths | Every build | < 2 minutes |
| **Integration** | External touchpoints | Every PR | < 5 minutes |
| **Contract** | All public APIs & events | Every PR | < 1 minute |
| **E2E** | 5-10 critical journeys | Nightly / pre-deploy | < 15 minutes |

---

## 🔬 Pattern 1: Unit Testing

> **Scope:** Single class, mocked dependencies, no Spring context, no I/O.
> **Speed:** Milliseconds per test.

### Structure: Arrange-Act-Assert (AAA)

```java
class PaymentValidatorTest {

    // ── System under test ──
    private PaymentValidator validator;

    // ── Dependencies (mocked) ──
    @Mock private AccountRepository accountRepository;
    @Mock private FraudRuleEngine fraudRuleEngine;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        validator = new PaymentValidator(accountRepository, fraudRuleEngine);
    }

    @Test
    @DisplayName("Should reject payment when amount exceeds daily limit")
    void shouldRejectPayment_whenAmountExceedsDailyLimit() {
        // ── Arrange ──
        var payment = PaymentFixtures.aPayment()
            .withAmount(Money.of("15000", "USD"))
            .build();

        when(accountRepository.findById("acc-123"))
            .thenReturn(Optional.of(AccountFixtures.activeAccount()));
        when(fraudRuleEngine.evaluate(any()))
            .thenReturn(FraudResult.clean());

        // ── Act ──
        var result = validator.validate(payment);

        // ── Assert ──
        assertThat(result.isValid()).isFalse();
        assertThat(result.errors())
            .hasSize(1)
            .first()
            .satisfies(error -> {
                assertThat(error.code()).isEqualTo("DAILY_LIMIT_EXCEEDED");
                assertThat(error.field()).isEqualTo("amount");
            });

        // Verify fraud engine was never called (short-circuit)
        verify(fraudRuleEngine, never()).evaluate(any());
    }
}
```

### Test Naming Convention

```
Method naming pattern:
  should<ExpectedBehavior>_when<Condition>

Examples:
  shouldRejectPayment_whenAmountIsNegative()
  shouldRetryThreeTimes_whenGatewayReturns503()
  shouldRouteToDlt_whenDeserializationFails()
  shouldCalculateFee_whenPaymentTypeIsCreditCard()
  shouldEmitMetric_whenCircuitBreakerOpens()

@DisplayName for human-readable output:
  @DisplayName("Should reject payment when amount is negative")
  @DisplayName("Should retry 3 times when gateway returns 503")
```

### Parameterized Tests

```java
class FeeCalculatorTest {

    @ParameterizedTest(name = "Fee for {0} on ${1} should be ${2}")
    @CsvSource({
        "CREDIT_CARD,  100.00,  2.90",
        "CREDIT_CARD, 1000.00, 29.00",
        "ACH,          100.00,  0.50",
        "ACH,         5000.00,  0.50",
        "WIRE,         100.00, 25.00",
        "WIRE,       50000.00, 25.00"
    })
    void shouldCalculateCorrectFee(PaymentType type, BigDecimal amount, BigDecimal expectedFee) {
        var calculator = new FeeCalculator();
        assertThat(calculator.calculate(type, amount))
            .isEqualByComparingTo(expectedFee);
    }

    @ParameterizedTest(name = "Should reject: {0}")
    @MethodSource("invalidPaymentProvider")
    void shouldRejectInvalidPayments(Payment payment, String expectedError) {
        var result = validator.validate(payment);
        assertThat(result.errors()).extracting("code").contains(expectedError);
    }

    static Stream<Arguments> invalidPaymentProvider() {
        return Stream.of(
            Arguments.of(aPayment().withAmount(null).build(), "AMOUNT_REQUIRED"),
            Arguments.of(aPayment().withAmount(Money.ZERO).build(), "AMOUNT_MUST_BE_POSITIVE"),
            Arguments.of(aPayment().withCurrency("XXX").build(), "INVALID_CURRENCY"),
            Arguments.of(aPayment().withAccountId("").build(), "ACCOUNT_REQUIRED")
        );
    }
}
```

### Testing Records and Sealed Types

```java
class PaymentEventTest {

    @Test
    void shouldCreateValidInitiatedEvent() {
        var event = new PaymentEvent.Initiated(
            "pay-123", Instant.now(), Money.of("100", "USD"), "cust-456");

        assertThat(event.paymentId()).isEqualTo("pay-123");
        assertThat(event).isInstanceOf(PaymentEvent.class);
    }

    @Test
    void shouldRejectInvalidAmountInRecord() {
        assertThatThrownBy(() -> new PaymentAmount(new BigDecimal("-10"), "USD"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("negative");
    }

    // Exhaustive switch coverage
    @ParameterizedTest
    @MethodSource("allPaymentEvents")
    void shouldHandleAllEventTypes(PaymentEvent event) {
        // Ensures our handler covers all sealed subtypes
        String result = eventHandler.describe(event);
        assertThat(result).isNotBlank();
    }

    static Stream<PaymentEvent> allPaymentEvents() {
        return Stream.of(
            new PaymentEvent.Initiated("p1", Instant.now(), Money.of("100", "USD"), "c1"),
            new PaymentEvent.Authorized("p1", Instant.now(), "AUTH-001"),
            new PaymentEvent.Captured("p1", Instant.now(), Money.of("100", "USD")),
            new PaymentEvent.Failed("p1", Instant.now(), "TIMEOUT", "Gateway timeout"),
            new PaymentEvent.Refunded("p1", Instant.now(), Money.of("100", "USD"), "Customer request")
        );
    }
}
```

### Testing with Mockito — Best Practices

```java
class PaymentServiceTest {

    // ── Strict stubbing (default in Mockito 4+) ──
    // Unused stubs cause test failure — keeps tests clean
    @Mock private PaymentGateway gateway;
    @Mock private PaymentRepository repository;

    // ── Argument captors for complex verification ──
    @Captor private ArgumentCaptor<PaymentEvent> eventCaptor;

    @Test
    void shouldPublishEventWithCorrectPayload() {
        // Arrange
        when(gateway.charge(any())).thenReturn(ChargeResult.success("tx-789"));

        // Act
        service.processPayment(aPayment().build());

        // Assert — capture what was published
        verify(eventPublisher).publish(eventCaptor.capture());
        var published = eventCaptor.getValue();
        assertThat(published).isInstanceOf(PaymentEvent.Captured.class);
        assertThat(((PaymentEvent.Captured) published).settlementId()).isEqualTo("tx-789");
    }

    // ── Verify interaction order ──
    @Test
    void shouldValidateBeforeCharging() {
        when(gateway.charge(any())).thenReturn(ChargeResult.success("tx-1"));

        service.processPayment(aPayment().build());

        var inOrder = inOrder(validator, gateway, repository);
        inOrder.verify(validator).validate(any());
        inOrder.verify(gateway).charge(any());
        inOrder.verify(repository).save(any());
    }

    // ── Test exception scenarios ──
    @Test
    void shouldNotSaveWhenGatewayFails() {
        when(gateway.charge(any())).thenThrow(new GatewayTimeoutException("timeout"));

        assertThatThrownBy(() -> service.processPayment(aPayment().build()))
            .isInstanceOf(PaymentProcessingException.class);

        verify(repository, never()).save(any());
    }
}
```

---

## 🔗 Pattern 2: Integration Testing with Testcontainers

> **Scope:** Your code + real external dependency (MongoDB, Kafka, Artemis, Redis).
> **Speed:** Seconds per test, container startup amortized.

### Testcontainers Base Configuration

```java
// Shared base class — containers started ONCE for entire test suite
@Testcontainers
@SpringBootTest
@ActiveProfiles("integration-test")
public abstract class IntegrationTestBase {

    // ── MongoDB ──
    @Container
    static final MongoDBContainer MONGO = new MongoDBContainer("mongo:7.0")
        .withReuse(true);     // Reuse across test runs (dev speed)

    // ── Kafka ──
    @Container
    static final KafkaContainer KAFKA = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
        .withKraft()          // No ZooKeeper
        .withReuse(true);

    // ── ActiveMQ Artemis ──
    @Container
    static final GenericContainer<?> ARTEMIS = new GenericContainer<>(
        DockerImageName.parse("apache/activemq-artemis:2.33.0"))
        .withExposedPorts(61616, 8161)
        .withEnv("ARTEMIS_USER", "admin")
        .withEnv("ARTEMIS_PASSWORD", "admin")
        .withReuse(true);

    // ── Wire containers into Spring ──
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", MONGO::getReplicaSetUrl);
        registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
        registry.add("spring.artemis.broker-url",
            () -> "tcp://%s:%d".formatted(ARTEMIS.getHost(), ARTEMIS.getMappedPort(61616)));
    }

    @Autowired protected MongoTemplate mongoTemplate;
    @Autowired protected KafkaTemplate<String, Object> kafkaTemplate;

    @BeforeEach
    void cleanDatabase() {
        mongoTemplate.getCollectionNames()
            .forEach(name -> mongoTemplate.dropCollection(name));
    }
}
```

### MongoDB Integration Test

```java
class PaymentRepositoryIntegrationTest extends IntegrationTestBase {

    @Autowired private PaymentRepository paymentRepository;

    @Test
    @DisplayName("Should find payments by customer with pagination")
    void shouldFindPaymentsByCustomer() {
        // Arrange — insert real documents
        var payments = IntStream.rangeClosed(1, 25)
            .mapToObj(i -> PaymentFixtures.aPayment()
                .withId("pay-" + i)
                .withCustomerId("cust-100")
                .withAmount(Money.of(String.valueOf(i * 10), "USD"))
                .build())
            .toList();
        paymentRepository.saveAll(payments);

        // Act
        var page = paymentRepository.findByCustomerId(
            "cust-100", PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "amount")));

        // Assert
        assertThat(page.getTotalElements()).isEqualTo(25);
        assertThat(page.getContent()).hasSize(10);
        assertThat(page.getContent().getFirst().amount())
            .isEqualTo(Money.of("250", "USD")); // Highest amount first
    }

    @Test
    @DisplayName("Should support MongoDB aggregation pipeline")
    void shouldAggregatePaymentsByStatus() {
        // Arrange
        paymentRepository.saveAll(List.of(
            aPayment().withStatus(COMPLETED).withAmount(Money.of("100", "USD")).build(),
            aPayment().withStatus(COMPLETED).withAmount(Money.of("200", "USD")).build(),
            aPayment().withStatus(FAILED).withAmount(Money.of("50", "USD")).build()
        ));

        // Act
        var results = paymentRepository.aggregateByStatus();

        // Assert
        assertThat(results)
            .hasSize(2)
            .anySatisfy(r -> {
                assertThat(r.status()).isEqualTo(COMPLETED);
                assertThat(r.totalAmount()).isEqualByComparingTo("300");
                assertThat(r.count()).isEqualTo(2);
            });
    }

    @Test
    @DisplayName("Should enforce unique constraint on idempotency key")
    void shouldRejectDuplicateIdempotencyKey() {
        var payment1 = aPayment().withIdempotencyKey("idem-1").build();
        var payment2 = aPayment().withIdempotencyKey("idem-1").build();

        paymentRepository.save(payment1);

        assertThatThrownBy(() -> paymentRepository.save(payment2))
            .isInstanceOf(DuplicateKeyException.class);
    }
}
```

### Kafka Integration Test

```java
class PaymentEventConsumerIntegrationTest extends IntegrationTestBase {

    @Autowired private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    @Autowired private PaymentRepository paymentRepository;

    // Wait for async consumer processing
    @Autowired private CountDownLatch processingLatch;  // or use Awaitility

    @Test
    @DisplayName("Should process payment event and persist to MongoDB")
    void shouldConsumeAndPersistPayment() {
        // Arrange
        var event = new PaymentEvent.Initiated(
            "pay-999", Instant.now(), Money.of("250", "USD"), "cust-123");

        // Act — publish to real Kafka
        kafkaTemplate.send("payments.transaction.initiated", "pay-999", event)
            .get(10, TimeUnit.SECONDS);

        // Assert — wait for consumer to process
        await().atMost(Duration.ofSeconds(30))
            .pollInterval(Duration.ofMillis(500))
            .untilAsserted(() -> {
                var payment = paymentRepository.findById("pay-999");
                assertThat(payment).isPresent();
                assertThat(payment.get().status()).isEqualTo(PaymentStatus.INITIATED);
                assertThat(payment.get().amount()).isEqualTo(Money.of("250", "USD"));
            });
    }

    @Test
    @DisplayName("Should send failed event to DLT after retries exhausted")
    void shouldRouteToDltAfterRetryExhaustion() {
        // Arrange — event that will cause processing failure
        var poisonEvent = new PaymentEvent.Initiated(
            "pay-poison", Instant.now(), Money.of("-100", "USD"), "cust-bad");

        // Set up DLT consumer
        var dltRecords = new CopyOnWriteArrayList<ConsumerRecord<String, Object>>();
        var dltConsumer = createConsumer("payments.transaction.initiated.dlt", dltRecords::add);

        // Act
        kafkaTemplate.send("payments.transaction.initiated", "pay-poison", poisonEvent);

        // Assert — should land in DLT
        await().atMost(Duration.ofSeconds(60))
            .untilAsserted(() -> {
                assertThat(dltRecords).hasSize(1);
                assertThat(dltRecords.getFirst().key()).isEqualTo("pay-poison");
            });

        // Verify it was NOT persisted to MongoDB
        assertThat(paymentRepository.findById("pay-poison")).isEmpty();
    }

    @Test
    @DisplayName("Should maintain ordering within partition for same key")
    void shouldMaintainOrderingPerKey() {
        var events = List.of(
            new PaymentEvent.Initiated("pay-order", Instant.now(), Money.of("100", "USD"), "c1"),
            new PaymentEvent.Authorized("pay-order", Instant.now(), "AUTH-1"),
            new PaymentEvent.Captured("pay-order", Instant.now(), Money.of("100", "USD"))
        );

        // Send sequentially with same key
        for (var event : events) {
            kafkaTemplate.send("payments.transaction.all", "pay-order", event)
                .get(5, TimeUnit.SECONDS);
        }

        // Verify processing order
        await().atMost(Duration.ofSeconds(30))
            .untilAsserted(() -> {
                var auditLog = auditRepository.findByPaymentId("pay-order");
                assertThat(auditLog).hasSize(3);
                assertThat(auditLog.stream().map(AuditEntry::eventType).toList())
                    .containsExactly("Initiated", "Authorized", "Captured");
            });
    }
}
```

### JMS / ActiveMQ Artemis Integration Test

```java
class JmsPaymentListenerIntegrationTest extends IntegrationTestBase {

    @Autowired private JmsTemplate jmsTemplate;
    @Autowired private PaymentRepository paymentRepository;

    @Test
    @DisplayName("Should consume JMS message and process payment")
    void shouldConsumeJmsMessage() {
        // Arrange
        var message = new JmsPaymentMessage("pay-jms-1", "100.00", "USD", "cust-1");

        // Act — send to real Artemis
        jmsTemplate.convertAndSend("payment.inbound.queue", message);

        // Assert
        await().atMost(Duration.ofSeconds(15))
            .untilAsserted(() -> {
                var payment = paymentRepository.findById("pay-jms-1");
                assertThat(payment).isPresent();
            });
    }

    @Test
    @DisplayName("Should move poison message to DLQ")
    void shouldMoveToDeadLetterQueue() {
        // Send malformed message
        jmsTemplate.send("payment.inbound.queue", session -> {
            var msg = session.createTextMessage("NOT_VALID_JSON{{{");
            return msg;
        });

        // Should land in Artemis DLQ
        await().atMost(Duration.ofSeconds(15))
            .untilAsserted(() -> {
                var dlqMessage = jmsTemplate.receive("DLQ.payment.inbound.queue");
                assertThat(dlqMessage).isNotNull();
            });
    }
}
```

---

## 📜 Pattern 3: Contract Testing

> **Scope:** Verify that producer and consumer agree on API/event schemas.
> **Why:** Prevents "works on my machine" integration failures.

### REST API Contract Test (Provider Side)

```java
// Using Spring Cloud Contract or Pact
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class PaymentApiContractTest {

    @Autowired private TestRestTemplate restTemplate;
    @MockBean private PaymentService paymentService;

    @Test
    @DisplayName("POST /payments should return 201 with payment ID")
    void shouldCreatePayment() {
        when(paymentService.create(any())).thenReturn(
            new PaymentResult("pay-new-1", PaymentStatus.INITIATED));

        var request = """
            {
                "amount": "100.00",
                "currency": "USD",
                "customerId": "cust-123",
                "idempotencyKey": "idem-abc"
            }
            """;

        var response = restTemplate.postForEntity("/api/v1/payments",
            new HttpEntity<>(request, jsonHeaders()), Map.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody())
            .containsKey("paymentId")
            .containsEntry("status", "INITIATED");
    }

    @Test
    @DisplayName("POST /payments should return 400 for invalid request")
    void shouldReturn400ForInvalidRequest() {
        var request = """
            {
                "amount": "-50",
                "currency": "INVALID"
            }
            """;

        var response = restTemplate.postForEntity("/api/v1/payments",
            new HttpEntity<>(request, jsonHeaders()), Map.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody())
            .containsKey("errors");
    }

    @Test
    @DisplayName("GET /payments/{id} should return 404 for non-existent payment")
    void shouldReturn404ForMissingPayment() {
        when(paymentService.findById("non-existent"))
            .thenReturn(Optional.empty());

        var response = restTemplate.getForEntity("/api/v1/payments/non-existent", Map.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

### Kafka Event Contract Test

```java
class PaymentEventContractTest {

    private final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    @Test
    @DisplayName("PaymentEvent.Initiated should serialize with all required fields")
    void shouldSerializeInitiatedEvent() throws Exception {
        var event = new PaymentEvent.Initiated(
            "pay-1", Instant.parse("2026-01-15T10:30:00Z"),
            Money.of("100.00", "USD"), "cust-1");

        String json = mapper.writeValueAsString(event);

        // Verify required fields exist in serialized form
        var tree = mapper.readTree(json);
        assertThat(tree.has("paymentId")).isTrue();
        assertThat(tree.has("timestamp")).isTrue();
        assertThat(tree.has("amount")).isTrue();
        assertThat(tree.has("customerId")).isTrue();
    }

    @Test
    @DisplayName("Should deserialize event from older schema version (backward compatibility)")
    void shouldHandleOlderSchemaVersion() throws Exception {
        // Simulates a message from an older producer (missing new optional fields)
        String oldSchemaJson = """
            {
                "paymentId": "pay-old",
                "timestamp": "2026-01-15T10:30:00Z",
                "amount": {"value": "100.00", "currency": "USD"},
                "customerId": "cust-1"
            }
            """;

        var event = mapper.readValue(oldSchemaJson, PaymentEvent.Initiated.class);

        assertThat(event.paymentId()).isEqualTo("pay-old");
        assertThat(event.amount().value()).isEqualByComparingTo("100.00");
    }

    @Test
    @DisplayName("Should tolerate unknown fields from newer schema version (forward compatibility)")
    void shouldTolerateNewFields() throws Exception {
        String newSchemaJson = """
            {
                "paymentId": "pay-new",
                "timestamp": "2026-01-15T10:30:00Z",
                "amount": {"value": "100.00", "currency": "USD"},
                "customerId": "cust-1",
                "riskScore": 42,
                "metadata": {"source": "mobile"}
            }
            """;

        // Should NOT throw even though riskScore and metadata don't exist in our schema
        var event = mapper.readValue(newSchemaJson, PaymentEvent.Initiated.class);
        assertThat(event.paymentId()).isEqualTo("pay-new");
    }
}
```

---

## 🎭 Pattern 4: BDD-Style Testing

> **When:** Complex business logic with stakeholder-readable scenarios.

### Nested Test Structure (Given-When-Then)

```java
@DisplayName("Payment Processing")
class PaymentProcessingTest {

    @Nested
    @DisplayName("Given a valid credit card payment")
    class ValidCreditCardPayment {

        private Payment payment;

        @BeforeEach
        void setUp() {
            payment = aPayment()
                .withType(PaymentType.CREDIT_CARD)
                .withAmount(Money.of("100", "USD"))
                .withAccountId("acc-active")
                .build();

            when(accountRepository.findById("acc-active"))
                .thenReturn(Optional.of(activeAccount()));
            when(fraudService.check(any()))
                .thenReturn(FraudResult.clean());
        }

        @Nested
        @DisplayName("When the gateway approves")
        class GatewayApproves {

            @BeforeEach
            void setUp() {
                when(gateway.charge(any()))
                    .thenReturn(ChargeResult.success("tx-100"));
            }

            @Test
            @DisplayName("Then payment status should be COMPLETED")
            void shouldCompletePayment() {
                var result = service.process(payment);
                assertThat(result.status()).isEqualTo(PaymentStatus.COMPLETED);
            }

            @Test
            @DisplayName("Then a Captured event should be published")
            void shouldPublishCapturedEvent() {
                service.process(payment);
                verify(eventPublisher).publish(argThat(event ->
                    event instanceof PaymentEvent.Captured c
                        && c.settlementId().equals("tx-100")));
            }

            @Test
            @DisplayName("Then fee should be 2.9%")
            void shouldChargeCreditCardFee() {
                var result = service.process(payment);
                assertThat(result.fee()).isEqualByComparingTo("2.90");
            }
        }

        @Nested
        @DisplayName("When the gateway times out")
        class GatewayTimesOut {

            @BeforeEach
            void setUp() {
                when(gateway.charge(any()))
                    .thenThrow(new GatewayTimeoutException("Connection timed out"));
            }

            @Test
            @DisplayName("Then payment should be PENDING_RETRY")
            void shouldMarkPendingRetry() {
                var result = service.process(payment);
                assertThat(result.status()).isEqualTo(PaymentStatus.PENDING_RETRY);
            }

            @Test
            @DisplayName("Then a retry event should be queued")
            void shouldQueueRetry() {
                service.process(payment);
                verify(retryQueue).enqueue(argThat(msg ->
                    msg.paymentId().equals(payment.id())
                        && msg.attemptNumber() == 1));
            }
        }
    }

    @Nested
    @DisplayName("Given a payment from a suspended account")
    class SuspendedAccount {

        @Test
        @DisplayName("Should reject immediately without calling gateway")
        void shouldRejectWithoutGatewayCall() {
            var payment = aPayment().withAccountId("acc-suspended").build();
            when(accountRepository.findById("acc-suspended"))
                .thenReturn(Optional.of(suspendedAccount()));

            var result = service.process(payment);

            assertThat(result.status()).isEqualTo(PaymentStatus.REJECTED);
            verify(gateway, never()).charge(any());
        }
    }
}
```

---

## 🏭 Pattern 5: Test Fixtures & Builders

> **Rule:** Never construct test data with raw constructors scattered across tests. Centralize in builders.

### Builder Pattern for Test Data

```java
public class PaymentFixtures {

    // ── Entry point — sensible defaults ──
    public static PaymentBuilder aPayment() {
        return new PaymentBuilder()
            .withId("pay-" + UUID.randomUUID().toString().substring(0, 8))
            .withAmount(Money.of("100.00", "USD"))
            .withType(PaymentType.CREDIT_CARD)
            .withCustomerId("cust-default")
            .withAccountId("acc-default")
            .withIdempotencyKey(UUID.randomUUID().toString())
            .withStatus(PaymentStatus.INITIATED)
            .withCreatedAt(Instant.now());
    }

    // ── Named presets for common scenarios ──
    public static PaymentBuilder aHighValuePayment() {
        return aPayment().withAmount(Money.of("50000", "USD"));
    }

    public static PaymentBuilder anAchPayment() {
        return aPayment().withType(PaymentType.ACH);
    }

    public static PaymentBuilder aFailedPayment() {
        return aPayment().withStatus(PaymentStatus.FAILED);
    }

    // ── Builder class ──
    public static class PaymentBuilder {
        private String id;
        private Money amount;
        private PaymentType type;
        private String customerId;
        private String accountId;
        private String idempotencyKey;
        private PaymentStatus status;
        private Instant createdAt;

        public PaymentBuilder withId(String id) { this.id = id; return this; }
        public PaymentBuilder withAmount(Money amount) { this.amount = amount; return this; }
        public PaymentBuilder withType(PaymentType type) { this.type = type; return this; }
        public PaymentBuilder withCustomerId(String id) { this.customerId = id; return this; }
        public PaymentBuilder withAccountId(String id) { this.accountId = id; return this; }
        public PaymentBuilder withIdempotencyKey(String key) { this.idempotencyKey = key; return this; }
        public PaymentBuilder withStatus(PaymentStatus status) { this.status = status; return this; }
        public PaymentBuilder withCreatedAt(Instant at) { this.createdAt = at; return this; }

        public Payment build() {
            return new Payment(id, amount, type, customerId, accountId,
                idempotencyKey, status, createdAt);
        }
    }
}

// ── Account fixtures ──
public class AccountFixtures {
    public static Account activeAccount() {
        return new Account("acc-default", AccountStatus.ACTIVE,
            Money.of("10000", "USD"), "cust-default");
    }

    public static Account suspendedAccount() {
        return new Account("acc-suspended", AccountStatus.SUSPENDED,
            Money.of("0", "USD"), "cust-suspended");
    }
}
```

### Fixture Organization

```
src/test/java/
  com.example.payment/
    fixtures/
      PaymentFixtures.java        ← Payment builders
      AccountFixtures.java        ← Account builders
      KafkaFixtures.java          ← ConsumerRecord builders
      JmsFixtures.java            ← JMS message builders
    base/
      IntegrationTestBase.java    ← Testcontainers setup
      WebTestBase.java            ← MockMvc setup
    unit/
      PaymentValidatorTest.java
      FeeCalculatorTest.java
    integration/
      PaymentRepositoryIT.java
      KafkaConsumerIT.java
    contract/
      PaymentApiContractTest.java
      PaymentEventContractTest.java
    e2e/
      PaymentJourneyE2ETest.java
```

---

## 🛡️ Pattern 6: Resilience Testing

> **Scope:** Verify that circuit breakers, retries, timeouts, and fallbacks work correctly.

### Circuit Breaker Test

```java
class CircuitBreakerTest {

    @Test
    @DisplayName("Should open circuit after 5 failures and return fallback")
    void shouldOpenCircuitAfterThreshold() {
        var circuitBreaker = CircuitBreaker.of("test", CircuitBreakerConfig.custom()
            .slidingWindowSize(5)
            .minimumNumberOfCalls(5)
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(10))
            .build());

        Supplier<String> decorated = CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> { throw new RuntimeException("Service down"); });

        // Trip the circuit
        for (int i = 0; i < 5; i++) {
            assertThatThrownBy(decorated::get).isInstanceOf(RuntimeException.class);
        }

        // Circuit should be OPEN now
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // Next call should fail fast with CallNotPermittedException
        assertThatThrownBy(decorated::get)
            .isInstanceOf(CallNotPermittedException.class);
    }

    @Test
    @DisplayName("Should transition to half-open and recover")
    void shouldRecoverAfterWaitDuration() {
        var circuitBreaker = CircuitBreaker.of("test", CircuitBreakerConfig.custom()
            .slidingWindowSize(5)
            .minimumNumberOfCalls(5)
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(1))
            .permittedNumberOfCallsInHalfOpenState(2)
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .build());

        var callCount = new AtomicInteger(0);
        Supplier<String> decorated = CircuitBreaker.decorateSupplier(circuitBreaker, () -> {
            if (callCount.incrementAndGet() <= 5) throw new RuntimeException("fail");
            return "recovered";
        });

        // Trip the circuit
        for (int i = 0; i < 5; i++) {
            assertThatThrownBy(decorated::get);
        }
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);

        // Wait for transition
        await().atMost(Duration.ofSeconds(3))
            .until(() -> circuitBreaker.getState() == CircuitBreaker.State.HALF_OPEN);

        // Recovery calls should succeed
        assertThat(decorated.get()).isEqualTo("recovered");
    }
}
```

### Retry Test

```java
class RetryBehaviorTest {

    @Test
    @DisplayName("Should retry 3 times with exponential backoff then succeed")
    void shouldRetryAndSucceed() {
        var callCount = new AtomicInteger(0);

        var retry = Retry.of("test", RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(100))
            .retryOnException(e -> e instanceof ConnectException)
            .build());

        Supplier<String> decorated = Retry.decorateSupplier(retry, () -> {
            if (callCount.incrementAndGet() < 3) {
                throw new ConnectException("Connection refused");
            }
            return "success";
        });

        assertThat(decorated.get()).isEqualTo("success");
        assertThat(callCount.get()).isEqualTo(3);
    }

    @Test
    @DisplayName("Should NOT retry on non-retryable exception")
    void shouldNotRetryOnValidationError() {
        var callCount = new AtomicInteger(0);

        var retry = Retry.of("test", RetryConfig.custom()
            .maxAttempts(3)
            .ignoreExceptions(ValidationException.class)
            .build());

        Supplier<String> decorated = Retry.decorateSupplier(retry, () -> {
            callCount.incrementAndGet();
            throw new ValidationException("Invalid amount");
        });

        assertThatThrownBy(decorated::get).isInstanceOf(ValidationException.class);
        assertThat(callCount.get()).isEqualTo(1); // Called only once — no retry
    }
}
```

### Timeout Test

```java
@Test
@DisplayName("Should timeout and cancel when service is slow")
void shouldTimeoutSlowService() {
    var timeLimiter = TimeLimiter.of(TimeLimiterConfig.custom()
        .timeoutDuration(Duration.ofMillis(500))
        .cancelRunningFuture(true)
        .build());

    var future = CompletableFuture.supplyAsync(() -> {
        try { Thread.sleep(5000); } catch (InterruptedException e) { }
        return "too late";
    });

    assertThatThrownBy(() -> timeLimiter.executeFutureSupplier(() -> future))
        .isInstanceOf(TimeoutException.class);
}
```

---

## 🔄 Pattern 7: Saga / Orchestration Testing

> **Scope:** Verify multi-step workflows with compensation logic.

```java
class PaymentSagaTest {

    @Mock private AccountService accountService;
    @Mock private FraudService fraudService;
    @Mock private PaymentGateway gateway;

    private PaymentSagaOrchestrator saga;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        saga = new PaymentSagaOrchestrator(accountService, fraudService, gateway);
    }

    @Test
    @DisplayName("Happy path: all steps succeed")
    void shouldCompleteAllSteps() {
        when(accountService.reserveFunds(any())).thenReturn(new Reservation("res-1"));
        when(fraudService.check(any())).thenReturn(FraudResult.approved());
        when(gateway.charge(any())).thenReturn(ChargeResult.success("tx-1"));

        var result = saga.execute(aPayment().build());

        assertThat(result.status()).isEqualTo(PaymentStatus.COMPLETED);
        var inOrder = inOrder(accountService, fraudService, gateway);
        inOrder.verify(accountService).reserveFunds(any());
        inOrder.verify(fraudService).check(any());
        inOrder.verify(gateway).charge(any());
    }

    @Test
    @DisplayName("Should compensate all completed steps when gateway fails")
    void shouldCompensateOnGatewayFailure() {
        var reservation = new Reservation("res-1");
        when(accountService.reserveFunds(any())).thenReturn(reservation);
        when(fraudService.check(any())).thenReturn(FraudResult.approved());
        when(gateway.charge(any())).thenThrow(new GatewayException("Declined"));

        var result = saga.execute(aPayment().build());

        assertThat(result.status()).isEqualTo(PaymentStatus.FAILED);

        // Compensation should happen in REVERSE order
        var inOrder = inOrder(gateway, fraudService, accountService);
        inOrder.verify(fraudService).clearFlag(any());
        inOrder.verify(accountService).releaseReservation("res-1");
    }

    @Test
    @DisplayName("Should NOT compensate steps after the failure point")
    void shouldNotCompensateUnexecutedSteps() {
        when(accountService.reserveFunds(any())).thenReturn(new Reservation("res-1"));
        when(fraudService.check(any())).thenReturn(FraudResult.rejected("HIGH_RISK"));

        var result = saga.execute(aPayment().build());

        // Gateway was never called, so no refund needed
        verify(gateway, never()).charge(any());
        verify(gateway, never()).refund(any());
        // Only reservation needs compensation
        verify(accountService).releaseReservation("res-1");
    }

    @Test
    @DisplayName("Should alert when compensation itself fails")
    void shouldAlertOnCompensationFailure() {
        when(accountService.reserveFunds(any())).thenReturn(new Reservation("res-1"));
        when(fraudService.check(any())).thenThrow(new ServiceException("Fraud service down"));
        doThrow(new ServiceException("Cannot release"))
            .when(accountService).releaseReservation(any());

        var result = saga.execute(aPayment().build());

        verify(alertService).critical(contains("compensation failure"), any());
    }
}
```

---

## 🏗️ Pattern 8: Architecture Testing with ArchUnit

> **Scope:** Enforce architectural rules automatically in CI.

```java
@AnalyzeClasses(packages = "com.example.payment", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    // ── Layer rules ──
    @ArchTest
    static final ArchRule domainShouldNotDependOnInfrastructure =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..infrastructure..", "..adapter..", "..config..");

    @ArchTest
    static final ArchRule domainShouldNotUseSpring =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..");

    // ── Naming conventions ──
    @ArchTest
    static final ArchRule controllersShouldEndWithController =
        classes().that().areAnnotatedWith(RestController.class)
            .should().haveSimpleNameEndingWith("Controller");

    @ArchTest
    static final ArchRule repositoriesShouldBeInterfaces =
        classes().that().haveSimpleNameEndingWith("Repository")
            .should().beInterfaces();

    // ── Dependency rules ──
    @ArchTest
    static final ArchRule servicesShouldNotDependOnControllers =
        noClasses().that().resideInAPackage("..service..")
            .should().dependOnClassesThat().resideInAPackage("..controller..");

    // ── Exception handling ──
    @ArchTest
    static final ArchRule noGenericExceptions =
        noClasses().should().throwGenericExceptions();

    // ── Field injection banned ──
    @ArchTest
    static final ArchRule noFieldInjection =
        noFields().should().beAnnotatedWith(Autowired.class)
            .because("Use constructor injection instead");

    // ── Records should be immutable (no setters) ──
    @ArchTest
    static final ArchRule recordsShouldNotHaveSetters =
        noMethods().that().areDeclaredInClassesThat().areRecords()
            .should().haveNameStartingWith("set");
}
```

---

## 🧬 Pattern 9: Mutation Testing

> **Purpose:** Verify your tests actually catch bugs, not just execute code.

### PIT Mutation Testing Configuration

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.payment.domain.*</param>
            <param>com.example.payment.service.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.payment.*Test</param>
        </targetTests>
        <mutators>
            <mutator>STRONGER</mutator>    <!-- More thorough mutations -->
        </mutators>
        <mutationThreshold>80</mutationThreshold>  <!-- Fail if < 80% killed -->
        <coverageThreshold>85</coverageThreshold>
        <timeoutConstant>3000</timeoutConstant>
        <excludedMethods>
            <param>toString</param>
            <param>hashCode</param>
        </excludedMethods>
    </configuration>
</plugin>
```

### Mutation Types to Watch For

```
SURVIVED mutations = weak tests!

Common survivors:
  • Boundary mutations (> → >=)        → Add boundary tests
  • Return value mutations (true → false) → Assert return values
  • Removed method calls               → Verify with Mockito
  • Negated conditionals               → Test both branches
  • Math mutations (+1 → -1)           → Assert exact calculations
  
Run: mvn org.pitest:pitest-maven:mutationCoverage
```

---

## ⚡ Pattern 10: Performance & Load Testing Patterns

### JMH Microbenchmarks

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class SerializationBenchmark {

    private ObjectMapper mapper;
    private PaymentEvent event;
    private byte[] serialized;

    @Setup
    public void setUp() throws Exception {
        mapper = new ObjectMapper().registerModule(new JavaTimeModule());
        event = new PaymentEvent.Initiated(
            "pay-1", Instant.now(), Money.of("100", "USD"), "cust-1");
        serialized = mapper.writeValueAsBytes(event);
    }

    @Benchmark
    public byte[] serialize() throws Exception {
        return mapper.writeValueAsBytes(event);
    }

    @Benchmark
    public PaymentEvent deserialize() throws Exception {
        return mapper.readValue(serialized, PaymentEvent.Initiated.class);
    }
}
```

### Concurrent Access Test

```java
@Test
@DisplayName("Should handle concurrent payments without data corruption")
void shouldHandleConcurrentPayments() throws Exception {
    int threadCount = 50;
    var latch = new CountDownLatch(threadCount);
    var errors = new CopyOnWriteArrayList<Throwable>();

    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < threadCount; i++) {
            final int idx = i;
            executor.submit(() -> {
                try {
                    var payment = aPayment()
                        .withId("concurrent-" + idx)
                        .withIdempotencyKey("idem-" + idx)
                        .build();
                    service.process(payment);
                } catch (Exception e) {
                    errors.add(e);
                } finally {
                    latch.countDown();
                }
            });
        }

        assertThat(latch.await(30, TimeUnit.SECONDS)).isTrue();
    }

    assertThat(errors).isEmpty();
    assertThat(paymentRepository.count()).isEqualTo(threadCount);
}

@Test
@DisplayName("Should be idempotent under concurrent duplicate submissions")
void shouldDeduplicateConcurrentDuplicates() throws Exception {
    int duplicateCount = 20;
    var latch = new CountDownLatch(duplicateCount);
    var results = new CopyOnWriteArrayList<PaymentResult>();

    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        for (int i = 0; i < duplicateCount; i++) {
            executor.submit(() -> {
                try {
                    var payment = aPayment()
                        .withId("same-payment")
                        .withIdempotencyKey("same-key")
                        .build();
                    results.add(service.process(payment));
                } catch (Exception e) {
                    // Expected for some duplicates
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(30, TimeUnit.SECONDS);
    }

    // Exactly ONE payment should exist
    assertThat(paymentRepository.findById("same-payment")).isPresent();
    // All successful results should have the same transaction ID
    var uniqueTxIds = results.stream()
        .filter(r -> r.status() == PaymentStatus.COMPLETED)
        .map(PaymentResult::transactionId)
        .distinct()
        .toList();
    assertThat(uniqueTxIds).hasSize(1);
}
```

---

## 📊 CI Pipeline Test Stages

```yaml
# .github/workflows/ci.yml (conceptual)
stages:
  # Stage 1: Fast feedback (< 2 min)
  compile-and-unit-test:
    - mvn compile
    - mvn test -Dgroups="unit"
    - mvn pitest:mutationCoverage      # Mutation testing
    - mvn archunit:check                # Architecture rules

  # Stage 2: Integration (< 5 min)
  integration-test:
    needs: compile-and-unit-test
    services: [docker]
    - mvn verify -Dgroups="integration" -Dtestcontainers=true

  # Stage 3: Contract (< 1 min)
  contract-test:
    needs: compile-and-unit-test
    - mvn test -Dgroups="contract"

  # Stage 4: E2E (< 15 min, nightly or pre-deploy)
  e2e-test:
    needs: [integration-test, contract-test]
    schedule: nightly
    - mvn verify -Dgroups="e2e" -Dtestcontainers=true
```

### JUnit 5 Tags for Test Categorization

```java
// Tag definitions
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Tag("unit")
public @interface UnitTest {}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Tag("integration")
public @interface IntegrationTest {}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Tag("contract")
public @interface ContractTest {}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Tag("e2e")
public @interface E2ETest {}

// Usage
@UnitTest
class PaymentValidatorTest { ... }

@IntegrationTest
class PaymentRepositoryIT { ... }
```

```xml
<!-- maven-surefire-plugin for unit tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <groups>unit</groups>
    </configuration>
</plugin>

<!-- maven-failsafe-plugin for integration tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <configuration>
        <groups>integration,contract</groups>
    </configuration>
</plugin>
```

---

## 🚫 Testing Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Testing implementation, not behavior** | Refactoring breaks tests even though behavior is correct | Assert outcomes, not internal method calls |
| **No assertion** | Test passes but verifies nothing | Every test MUST have at least one meaningful assertion |
| **Mocking the SUT** | You're testing the mock, not your code | Only mock dependencies, never the class under test |
| **Shared mutable state between tests** | Order-dependent, flaky tests | `@BeforeEach` reset, fresh fixtures per test |
| **Sleeping instead of awaiting** | Slow AND flaky | Use `Awaitility.await()` with assertions |
| **Catch-all exception handling** | Hides real failures | Let unexpected exceptions propagate |
| **Giant test methods** | Hard to diagnose failures | One logical assertion per test |
| **Testing getters/setters** | Zero value, clutters suite | Test behavior, not data access |
| **Mocking everything** | Integration bugs slip through | Use real dependencies where practical |
| **No negative tests** | Only happy path covered | Test invalid inputs, error paths, edge cases |
| **Ignoring flaky tests** (`@Disabled`) | Rot accumulates, real bugs hide | Fix or delete — disabled tests are dead code |
| **Copy-paste test data** | Inconsistent, hard to maintain | Use fixture builders |
| **Integration tests without cleanup** | Data leaks between tests | `@BeforeEach` cleanup or `@Transactional` |
| **Asserting on `toString()`** | Brittle — breaks on format changes | Assert on specific fields |

---

## 📏 Quick Reference: What to Test Where

| What | Unit | Integration | Contract | E2E |
|------|------|-------------|----------|-----|
| **Validation logic** | ✅ Primary | | | |
| **Business rules** | ✅ Primary | | | |
| **Fee calculations** | ✅ Primary | | | |
| **State machines** | ✅ Primary | | | |
| **Repository queries** | | ✅ Primary | | |
| **Kafka produce/consume** | | ✅ Primary | | |
| **JMS messaging** | | ✅ Primary | | |
| **MongoDB aggregations** | | ✅ Primary | | |
| **REST API responses** | | | ✅ Primary | |
| **Event schema compatibility** | | | ✅ Primary | |
| **Circuit breaker behavior** | ✅ Logic | ✅ Full | | |
| **Retry behavior** | ✅ Logic | ✅ Full | | |
| **Saga compensation** | ✅ Primary | | | |
| **Full payment journey** | | | | ✅ Primary |
| **Architecture rules** | ✅ ArchUnit | | | |
| **Concurrency safety** | ✅ + ✅ | ✅ | | |

---

## 💡 Golden Rules of Testing

```
1.  Test BEHAVIOR, not implementation — ask "what should happen" not "how does it work."
2.  One logical assertion per test — when it fails, you know EXACTLY what broke.
3.  Tests are documentation — a new developer should understand the system by reading tests.
4.  Fast tests run often — if your unit tests take > 30 seconds, something's wrong.
5.  Flaky tests are worse than no tests — they erode trust. Fix immediately or delete.
6.  Test the sad paths MORE than the happy paths — that's where production bugs live.
7.  Test data builders are not optional — they're required infrastructure.
8.  Every bug gets a regression test BEFORE the fix — prove the bug, then fix it.
9.  Mutation testing reveals the tests you think you have but don't.
10. If it's hard to test, the DESIGN is wrong — testability is a design quality signal.
```

---

*Last updated: February 2026 | Stack: Java 21+ / JUnit 5 / Spring Boot 3.x / Testcontainers / Mockito / ArchUnit / PIT*
