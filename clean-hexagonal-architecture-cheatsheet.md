# 🏛️ Clean Architecture & Hexagonal Architecture Cheat Sheet

> **Purpose:** Practical guide to structuring enterprise Java applications with Clean/Hexagonal Architecture. Reference before designing any new service, module, or package structure.
> **Stack context:** Java 21+ / Spring Boot 3.x / MongoDB / Kafka / ActiveMQ Artemis

---

## 📋 Architecture Decision Framework

Before structuring any service, answer:

| Question | Determines |
|----------|-----------|
| How **complex** is the domain logic? | Simple CRUD → skip; Rich domain → invest |
| How many **external systems** integrate? | More integrations → more ports |
| How likely are **technology swaps**? | DB migration, queue change → adapter isolation |
| How large is the **team**? | Small team → pragmatic layering; Large → strict boundaries |
| What's the **testing strategy**? | Domain testability drives architecture |
| Will this service **grow** significantly? | If yes, modular structure pays off early |

---

## 🔷 The Dependency Rule — Foundation of Everything

```
The ONE rule that governs all of Clean/Hexagonal Architecture:

     DEPENDENCIES POINT INWARD — ALWAYS

  ┌────────────────────────────────────────────────────────┐
  │  INFRASTRUCTURE (Frameworks, DB, Queue, HTTP)          │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  APPLICATION (Use Cases, Orchestration)           │  │
  │  │  ┌────────────────────────────────────────────┐  │  │
  │  │  │           DOMAIN                            │  │  │
  │  │  │  (Entities, Value Objects, Domain Events,   │  │  │
  │  │  │   Business Rules, Repository Interfaces)    │  │  │
  │  │  └────────────────────────────────────────────┘  │  │
  │  └──────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────┘

  • Domain knows NOTHING about application or infrastructure
  • Application knows domain, but NOT infrastructure specifics
  • Infrastructure knows both, but domain never knows infrastructure
  • Interfaces (ports) are defined by INNER layers
  • Implementations (adapters) live in OUTER layers
```

---

## ⬡ Pattern 1: Hexagonal Architecture (Ports & Adapters)

### Core Concept

```
The application is a HEXAGON:
  • INSIDE: Domain logic + application services
  • PORTS:  Interfaces on the hexagon's edges (defined by domain/application)
  • ADAPTERS: Concrete implementations that plug into ports

                        REST API Adapter
                             │
                    ┌────────▼────────┐
  Kafka Consumer ──►│                  │◄── Scheduled Jobs
  Adapter           │    APPLICATION   │    Adapter
                    │                  │
  JMS Listener ────►│  ┌────────────┐  │
  Adapter           │  │   DOMAIN   │  │◄── CLI Adapter
                    │  │            │  │
                    │  └────────────┘  │
                    │                  │
                    └──┬─────┬─────┬──┘
                       │     │     │
                       ▼     ▼     ▼
                    MongoDB Kafka  Artemis
                    Adapter Adapter Adapter

LEFT SIDE (Driving/Primary):        RIGHT SIDE (Driven/Secondary):
  Who CALLS us                        Who WE CALL
  • REST Controller                   • Database Repository
  • Kafka Consumer                    • Message Publisher
  • JMS Listener                      • External API Client
  • Scheduler                         • Cache
  • CLI                               • Email/SMS Service
```

### Port Types

```java
// ══════════════════════════════════════════════════════════
// DRIVING PORTS (Primary) — "How the outside world talks to us"
// Defined in APPLICATION layer, implemented by APPLICATION layer
// Called by INFRASTRUCTURE adapters (controllers, consumers)
// ══════════════════════════════════════════════════════════

public interface ProcessPaymentUseCase {
    PaymentResult process(ProcessPaymentCommand command);
}

public interface RefundPaymentUseCase {
    RefundResult refund(RefundPaymentCommand command);
}

public interface PaymentQueryPort {
    Optional<PaymentView> findById(String paymentId);
    Page<PaymentView> search(PaymentSearchCriteria criteria, Pageable pageable);
}


// ══════════════════════════════════════════════════════════
// DRIVEN PORTS (Secondary) — "How we talk to the outside world"
// Defined in DOMAIN or APPLICATION layer
// IMPLEMENTED by INFRASTRUCTURE adapters
// ══════════════════════════════════════════════════════════

// Defined in DOMAIN — core persistence abstraction
public interface PaymentRepository {
    Optional<Payment> findById(String id);
    Payment save(Payment payment);
    boolean existsByIdempotencyKey(String key);
}

// Defined in APPLICATION — event publishing abstraction
public interface PaymentEventPublisher {
    void publish(PaymentEvent event);
}

// Defined in APPLICATION — external service abstraction
public interface PaymentGatewayPort {
    ChargeResult charge(ChargeRequest request);
    RefundResult refund(String transactionId, Money amount);
}

public interface FraudCheckPort {
    FraudResult evaluate(FraudCheckRequest request);
}

public interface NotificationPort {
    void sendPaymentConfirmation(String customerId, PaymentConfirmation confirmation);
}
```

---

## 📁 Pattern 2: Package Structure

### Structure A: Package-by-Layer-and-Port (Recommended for Most Services)

```
com.example.payment/
│
├── domain/                          ← INNERMOST — zero dependencies
│   ├── model/
│   │   ├── Payment.java             ← Aggregate root
│   │   ├── PaymentStatus.java       ← Enum
│   │   ├── Money.java               ← Value object (record)
│   │   ├── FeeBreakdown.java        ← Value object (record)
│   │   └── PaymentMethod.java       ← Value object (record)
│   ├── event/
│   │   └── PaymentEvent.java        ← Sealed interface + records
│   ├── exception/
│   │   ├── PaymentNotFoundException.java
│   │   ├── InsufficientFundsException.java
│   │   └── DuplicatePaymentException.java
│   ├── repository/
│   │   └── PaymentRepository.java   ← Interface (driven port)
│   └── service/
│       ├── FeeCalculator.java        ← Domain service (pure logic)
│       └── PaymentValidator.java     ← Domain validation rules
│
├── application/                     ← USE CASES — orchestration
│   ├── port/
│   │   ├── in/                      ← Driving ports (interfaces)
│   │   │   ├── ProcessPaymentUseCase.java
│   │   │   ├── RefundPaymentUseCase.java
│   │   │   └── PaymentQueryPort.java
│   │   └── out/                     ← Driven ports (interfaces)
│   │       ├── PaymentGatewayPort.java
│   │       ├── FraudCheckPort.java
│   │       ├── PaymentEventPublisher.java
│   │       └── NotificationPort.java
│   ├── service/
│   │   ├── PaymentCommandService.java    ← Implements driving ports
│   │   └── PaymentQueryService.java      ← Implements query port
│   ├── dto/
│   │   ├── ProcessPaymentCommand.java    ← Input records
│   │   ├── RefundPaymentCommand.java
│   │   ├── PaymentResult.java            ← Output records
│   │   ├── PaymentView.java              ← Query results
│   │   └── PaymentSearchCriteria.java
│   └── saga/
│       └── PaymentSagaOrchestrator.java  ← Multi-step workflow
│
├── infrastructure/                  ← OUTERMOST — frameworks & adapters
│   ├── adapter/
│   │   ├── in/                      ← Driving adapters
│   │   │   ├── rest/
│   │   │   │   ├── PaymentController.java
│   │   │   │   ├── PaymentApiMapper.java     ← API ↔ Command mapping
│   │   │   │   ├── PaymentApiRequest.java    ← API request record
│   │   │   │   ├── PaymentApiResponse.java   ← API response record
│   │   │   │   └── GlobalExceptionHandler.java
│   │   │   ├── kafka/
│   │   │   │   └── PaymentEventConsumer.java
│   │   │   ├── jms/
│   │   │   │   └── PaymentJmsListener.java
│   │   │   └── scheduler/
│   │   │       └── OutboxPollerScheduler.java
│   │   └── out/                     ← Driven adapters
│   │       ├── persistence/
│   │       │   ├── MongoPaymentRepository.java   ← Implements PaymentRepository
│   │       │   ├── PaymentDocument.java          ← MongoDB document
│   │       │   └── PaymentDocumentMapper.java    ← Document ↔ Domain mapping
│   │       ├── messaging/
│   │       │   ├── KafkaPaymentEventPublisher.java ← Implements PaymentEventPublisher
│   │       │   └── ArtemisPaymentPublisher.java
│   │       ├── gateway/
│   │       │   ├── StripePaymentGateway.java     ← Implements PaymentGatewayPort
│   │       │   └── StripeApiMapper.java
│   │       ├── fraud/
│   │       │   └── InternalFraudService.java     ← Implements FraudCheckPort
│   │       └── notification/
│   │           └── EmailNotificationAdapter.java ← Implements NotificationPort
│   └── config/
│       ├── MongoConfig.java
│       ├── KafkaConfig.java
│       ├── ArtemisConfig.java
│       ├── SecurityConfig.java
│       └── ResilienceConfig.java
│
└── PaymentServiceApplication.java   ← Spring Boot entry point
```

### Structure B: Package-by-Feature (For Simpler Services)

```
com.example.payment/
│
├── payment/
│   ├── Payment.java                  ← Domain model
│   ├── PaymentStatus.java
│   ├── PaymentRepository.java        ← Port (interface)
│   ├── PaymentService.java           ← Use case
│   ├── PaymentController.java        ← Driving adapter
│   ├── MongoPaymentRepository.java   ← Driven adapter
│   └── PaymentEvent.java             ← Domain events
│
├── fraud/
│   ├── FraudCheckPort.java
│   ├── FraudCheckService.java
│   └── InternalFraudAdapter.java
│
├── gateway/
│   ├── PaymentGatewayPort.java
│   └── StripeGatewayAdapter.java
│
├── shared/
│   ├── Money.java
│   └── DateRange.java
│
└── config/
    └── ...
```

---

## 🧩 Pattern 3: Layer Implementation

### Domain Layer — Pure Business Logic

```java
// ═══ DOMAIN MODEL — NO framework imports ═══

// Aggregate root with business behavior
public class Payment {

    private String id;
    private String customerId;
    private Money amount;
    private PaymentStatus status;
    private String idempotencyKey;
    private FeeBreakdown fees;
    private GatewayResponse gatewayResponse;
    private List<StatusChange> statusHistory;
    private Instant createdAt;
    private Instant updatedAt;

    private final List<PaymentEvent> domainEvents = new ArrayList<>();

    // ── Factory method — controlled creation ──
    public static Payment initiate(String customerId, Money amount,
                                    PaymentMethod method, String idempotencyKey) {
        // Business validation at creation
        Objects.requireNonNull(customerId, "customerId required");
        Objects.requireNonNull(amount, "amount required");
        if (amount.isNegativeOrZero()) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        var payment = new Payment();
        payment.id = UUID.randomUUID().toString();
        payment.customerId = customerId;
        payment.amount = amount;
        payment.status = PaymentStatus.INITIATED;
        payment.idempotencyKey = idempotencyKey;
        payment.statusHistory = new ArrayList<>();
        payment.createdAt = Instant.now();
        payment.updatedAt = Instant.now();

        payment.addEvent(new PaymentEvent.Initiated(
            payment.id, Instant.now(), amount, customerId, method, idempotencyKey));

        return payment;
    }

    // ── Business behavior — state transitions with rules ──
    public void authorize(String authCode) {
        assertStatus(PaymentStatus.INITIATED, "authorize");
        transitionTo(PaymentStatus.AUTHORIZED, "Authorization received");
        this.gatewayResponse = new GatewayResponse(null, authCode, null);
        addEvent(new PaymentEvent.Authorized(id, Instant.now(), authCode, null));
    }

    public void capture(Money settledAmount, String transactionId, FeeBreakdown fees) {
        assertStatus(PaymentStatus.AUTHORIZED, "capture");
        if (settledAmount.compareTo(amount) > 0) {
            throw new IllegalArgumentException("Settled amount exceeds authorized amount");
        }
        this.fees = fees;
        this.gatewayResponse = new GatewayResponse(transactionId,
            gatewayResponse.authCode(), "SUCCESS");
        transitionTo(PaymentStatus.CAPTURED, "Payment captured");
        addEvent(new PaymentEvent.Captured(id, Instant.now(), settledAmount, transactionId, fees.total()));
    }

    public void fail(String errorCode, String reason, boolean retryable) {
        // Can fail from multiple states
        if (status.isTerminal()) {
            throw new IllegalStateException("Cannot fail from terminal status: " + status);
        }
        transitionTo(PaymentStatus.FAILED, reason);
        addEvent(new PaymentEvent.Failed(id, Instant.now(), errorCode, reason, retryable));
    }

    public void refund(Money refundAmount, String reason) {
        assertStatus(PaymentStatus.CAPTURED, "refund");
        if (refundAmount.compareTo(amount) > 0) {
            throw new IllegalArgumentException("Refund exceeds payment amount");
        }
        transitionTo(PaymentStatus.REFUNDED, reason);
        addEvent(new PaymentEvent.Refunded(id, Instant.now(), refundAmount,
            UUID.randomUUID().toString(), reason));
    }

    // ── Internal helpers ──
    private void transitionTo(PaymentStatus newStatus, String reason) {
        statusHistory.add(new StatusChange(status, newStatus, reason, Instant.now()));
        status = newStatus;
        updatedAt = Instant.now();
    }

    private void assertStatus(PaymentStatus expected, String operation) {
        if (status != expected) {
            throw new IllegalStateException(
                "Cannot %s: expected status %s but was %s".formatted(operation, expected, status));
        }
    }

    private void addEvent(PaymentEvent event) {
        domainEvents.add(event);
    }

    public List<PaymentEvent> domainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearEvents() {
        domainEvents.clear();
    }

    // Getters (no setters — state changes through business methods only)
    public String id() { return id; }
    public PaymentStatus status() { return status; }
    public Money amount() { return amount; }
    // ...
}

// ═══ DOMAIN VALUE OBJECTS — immutable records ═══
public record Money(BigDecimal value, String currency) implements Comparable<Money> {
    public Money {
        Objects.requireNonNull(value);
        Objects.requireNonNull(currency);
        value = value.setScale(2, RoundingMode.HALF_UP);
    }

    public static Money of(String amount, String currency) {
        return new Money(new BigDecimal(amount), currency);
    }

    public static final Money ZERO = new Money(BigDecimal.ZERO, "USD");

    public boolean isNegativeOrZero() { return value.compareTo(BigDecimal.ZERO) <= 0; }
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(value.add(other.value), currency);
    }
    public Money subtract(Money other) {
        assertSameCurrency(other);
        return new Money(value.subtract(other.value), currency);
    }
    public int compareTo(Money other) {
        assertSameCurrency(other);
        return value.compareTo(other.value);
    }
    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch: %s vs %s".formatted(currency, other.currency));
        }
    }
}

// ═══ DOMAIN SERVICE — pure logic, no I/O ═══
public class FeeCalculator {
    private static final Map<PaymentType, BigDecimal> FEE_RATES = Map.of(
        PaymentType.CREDIT_CARD, new BigDecimal("0.029"),
        PaymentType.ACH, new BigDecimal("0.001"),
        PaymentType.WIRE, new BigDecimal("0.0005")
    );

    public FeeBreakdown calculate(PaymentType type, Money amount) {
        var rate = FEE_RATES.getOrDefault(type, BigDecimal.ZERO);
        var processingFee = amount.value().multiply(rate).setScale(2, RoundingMode.HALF_UP);
        var platformFee = new BigDecimal("0.30");  // Fixed platform fee
        return new FeeBreakdown(processingFee, platformFee, processingFee.add(platformFee));
    }
}

// ═══ DOMAIN REPOSITORY PORT — defined by domain, implemented by infra ═══
public interface PaymentRepository {
    Optional<Payment> findById(String id);
    Payment save(Payment payment);
    boolean existsByIdempotencyKey(String key);
}
```

### Application Layer — Use Cases & Orchestration

```java
// ═══ DRIVING PORT — interface ═══
public interface ProcessPaymentUseCase {
    PaymentResult process(ProcessPaymentCommand command);
}

// ═══ COMMAND — immutable input ═══
public record ProcessPaymentCommand(
    String customerId,
    Money amount,
    PaymentType type,
    PaymentMethod method,
    String idempotencyKey,
    String correlationId
) {
    public ProcessPaymentCommand {
        Objects.requireNonNull(customerId, "customerId required");
        Objects.requireNonNull(amount, "amount required");
        Objects.requireNonNull(type, "type required");
        Objects.requireNonNull(idempotencyKey, "idempotencyKey required");
    }
}

// ═══ APPLICATION SERVICE — implements the use case ═══
@Service
public class PaymentCommandService implements ProcessPaymentUseCase, RefundPaymentUseCase {

    // All dependencies are PORTS (interfaces) — no concrete infrastructure
    private final PaymentRepository paymentRepository;
    private final PaymentGatewayPort gatewayPort;
    private final FraudCheckPort fraudCheckPort;
    private final PaymentEventPublisher eventPublisher;
    private final NotificationPort notificationPort;
    private final FeeCalculator feeCalculator;          // Domain service
    private final PaymentValidator paymentValidator;     // Domain service

    // Constructor injection — all ports
    public PaymentCommandService(
            PaymentRepository paymentRepository,
            PaymentGatewayPort gatewayPort,
            FraudCheckPort fraudCheckPort,
            PaymentEventPublisher eventPublisher,
            NotificationPort notificationPort) {
        this.paymentRepository = paymentRepository;
        this.gatewayPort = gatewayPort;
        this.fraudCheckPort = fraudCheckPort;
        this.eventPublisher = eventPublisher;
        this.notificationPort = notificationPort;
        this.feeCalculator = new FeeCalculator();        // Domain service — no injection needed
        this.paymentValidator = new PaymentValidator();
    }

    @Override
    public PaymentResult process(ProcessPaymentCommand command) {
        // 1. Idempotency check
        if (paymentRepository.existsByIdempotencyKey(command.idempotencyKey())) {
            return PaymentResult.duplicate(command.idempotencyKey());
        }

        // 2. Create domain aggregate
        var payment = Payment.initiate(
            command.customerId(), command.amount(), command.method(), command.idempotencyKey());

        // 3. Domain validation
        var validation = paymentValidator.validate(payment);
        if (!validation.isValid()) {
            return PaymentResult.validationFailed(validation.errors());
        }

        // 4. Fraud check (via port)
        var fraudResult = fraudCheckPort.evaluate(
            new FraudCheckRequest(command.customerId(), command.amount()));
        if (!fraudResult.approved()) {
            payment.fail("FRAUD_REJECTED", fraudResult.reason(), false);
            paymentRepository.save(payment);
            publishEvents(payment);
            return PaymentResult.rejected(fraudResult.reason());
        }

        // 5. Calculate fees (domain service)
        var fees = feeCalculator.calculate(command.type(), command.amount());

        // 6. Charge gateway (via port)
        try {
            var chargeResult = gatewayPort.charge(
                new ChargeRequest(payment.id(), command.amount(), command.method()));
            payment.authorize(chargeResult.authCode());
            payment.capture(command.amount(), chargeResult.transactionId(), fees);
        } catch (GatewayException e) {
            payment.fail(e.errorCode(), e.getMessage(), e.isRetryable());
            paymentRepository.save(payment);
            publishEvents(payment);
            return PaymentResult.failed(e.getMessage());
        }

        // 7. Persist
        paymentRepository.save(payment);

        // 8. Publish domain events
        publishEvents(payment);

        // 9. Notification (best effort — don't fail the payment if notification fails)
        try {
            notificationPort.sendPaymentConfirmation(
                command.customerId(), PaymentConfirmation.from(payment));
        } catch (Exception e) {
            // Log but don't fail — notification is non-critical
            log.warn("Notification failed for payment {}: {}", payment.id(), e.getMessage());
        }

        return PaymentResult.success(payment.id(), fees);
    }

    private void publishEvents(Payment payment) {
        payment.domainEvents().forEach(eventPublisher::publish);
        payment.clearEvents();
    }
}
```

### Infrastructure Layer — Adapters

```java
// ═══ DRIVING ADAPTER: REST Controller ═══
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    // Depends on USE CASE (driving port), not service implementation
    private final ProcessPaymentUseCase processPayment;
    private final RefundPaymentUseCase refundPayment;
    private final PaymentQueryPort paymentQuery;
    private final PaymentApiMapper mapper;

    @PostMapping
    public ResponseEntity<PaymentApiResponse> createPayment(
            @RequestBody @Valid PaymentApiRequest request) {

        // Map API DTO → Application command
        var command = mapper.toCommand(request);

        // Execute use case
        var result = processPayment.process(command);

        // Map Application result → API response
        return switch (result.status()) {
            case SUCCESS -> ResponseEntity.status(HttpStatus.CREATED)
                .body(mapper.toResponse(result));
            case DUPLICATE -> ResponseEntity.ok(mapper.toResponse(result));
            case VALIDATION_FAILED -> ResponseEntity.badRequest()
                .body(mapper.toErrorResponse(result));
            case REJECTED, FAILED -> ResponseEntity.unprocessableEntity()
                .body(mapper.toErrorResponse(result));
        };
    }
}

// ═══ DRIVING ADAPTER: Kafka Consumer ═══
@Component
public class PaymentEventConsumer {

    private final ProcessPaymentUseCase processPayment;  // Same port, different adapter

    @KafkaListener(topics = "payments.commands.process")
    public void onProcessPaymentCommand(PaymentCommandMessage message, Acknowledgment ack) {
        var command = new ProcessPaymentCommand(
            message.customerId(), message.amount(), message.type(),
            message.method(), message.idempotencyKey(), message.correlationId());

        processPayment.process(command);
        ack.acknowledge();
    }
}

// ═══ DRIVEN ADAPTER: MongoDB Repository ═══
@Repository
public class MongoPaymentRepository implements PaymentRepository {

    private final MongoTemplate mongoTemplate;
    private final PaymentDocumentMapper mapper;

    @Override
    public Optional<Payment> findById(String id) {
        var doc = mongoTemplate.findById(id, PaymentDocument.class);
        return Optional.ofNullable(doc).map(mapper::toDomain);
    }

    @Override
    public Payment save(Payment payment) {
        var doc = mapper.toDocument(payment);
        mongoTemplate.save(doc);
        return payment;
    }

    @Override
    public boolean existsByIdempotencyKey(String key) {
        return mongoTemplate.exists(
            Query.query(Criteria.where("idempotencyKey").is(key)),
            PaymentDocument.class
        );
    }
}

// ═══ DRIVEN ADAPTER: Kafka Event Publisher ═══
@Component
public class KafkaPaymentEventPublisher implements PaymentEventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final MongoTemplate mongoTemplate;  // For outbox

    @Override
    public void publish(PaymentEvent event) {
        // Outbox pattern — write to outbox for reliable delivery
        var outbox = OutboxEvent.from(event);
        mongoTemplate.insert(outbox);
        // Actual Kafka publish happens via OutboxPoller
    }
}

// ═══ DRIVEN ADAPTER: Stripe Gateway ═══
@Component
public class StripePaymentGateway implements PaymentGatewayPort {

    private final RestClient stripeClient;
    private final CircuitBreaker circuitBreaker;

    @Override
    public ChargeResult charge(ChargeRequest request) {
        return CircuitBreaker.decorateSupplier(circuitBreaker, () -> {
            var stripeRequest = toStripeRequest(request);
            var stripeResponse = stripeClient.post()
                .uri("/v1/charges")
                .body(stripeRequest)
                .retrieve()
                .body(StripeChargeResponse.class);
            return toChargeResult(stripeResponse);
        }).get();
    }
}
```

---

## 🗺️ Pattern 4: Mapping Between Layers

### Mapping Strategy

```
Each layer has its OWN data structures:

API Layer        Application Layer       Domain Layer        Infrastructure Layer
─────────       ─────────────────       ──────────         ────────────────────
PaymentApi      ProcessPayment          Payment             PaymentDocument
Request         Command                 (Aggregate)         (MongoDB doc)
     │               │                       │                    │
     └──► Mapper ────┘                       └──── Mapper ───────┘
     PaymentApiMapper                        PaymentDocumentMapper

WHY separate models per layer?
  • API changes don't ripple into domain
  • DB schema changes don't affect API
  • Domain model is pure business logic
  • Each layer optimized for its purpose
```

### Mapper Implementations

```java
// ── API ↔ Application ──
@Component
public class PaymentApiMapper {

    public ProcessPaymentCommand toCommand(PaymentApiRequest request) {
        return new ProcessPaymentCommand(
            request.customerId(),
            Money.of(request.amount(), request.currency()),
            PaymentType.valueOf(request.paymentType()),
            new PaymentMethod(request.method().type(), request.method().last4(), request.method().brand()),
            request.idempotencyKey(),
            MDC.get("correlationId")
        );
    }

    public PaymentApiResponse toResponse(PaymentResult result) {
        return new PaymentApiResponse(
            result.paymentId(),
            result.status().name(),
            result.fees() != null ? result.fees().total().toString() : null
        );
    }
}

// ── Domain ↔ Persistence ──
@Component
public class PaymentDocumentMapper {

    public Payment toDomain(PaymentDocument doc) {
        return Payment.reconstitute(    // Special factory for loading from persistence
            doc.id(),
            doc.customerId(),
            Money.of(doc.amount(), doc.currency()),
            PaymentStatus.valueOf(doc.status()),
            doc.idempotencyKey(),
            doc.statusHistory().stream()
                .map(h -> new StatusChange(
                    PaymentStatus.valueOf(h.from()),
                    PaymentStatus.valueOf(h.to()),
                    h.reason(), h.changedAt()))
                .toList(),
            doc.createdAt(),
            doc.version()
        );
    }

    public PaymentDocument toDocument(Payment payment) {
        return new PaymentDocument(
            payment.id(),
            payment.customerId(),
            payment.amount().value().toString(),
            payment.amount().currency(),
            payment.status().name(),
            payment.idempotencyKey(),
            // ... map all fields
            payment.createdAt(),
            payment.version()
        );
    }
}
```

---

## 🧪 Pattern 5: Testing by Layer

```
Domain Layer Tests:
  • Pure unit tests — NO mocks, NO Spring, NO I/O
  • Test business rules, state transitions, validations
  • Fastest tests in the suite
  
Application Layer Tests:
  • Unit tests with MOCKED ports
  • Test use case orchestration
  • Verify correct port interactions (order, arguments)

Infrastructure Adapter Tests:
  • Integration tests with REAL dependencies (Testcontainers)
  • Test MongoDB adapter with real MongoDB
  • Test Kafka adapter with real Kafka
  • Test REST controller with MockMvc or TestRestTemplate

End-to-End Tests:
  • Full Spring context, all real dependencies
  • Test complete flow from API call to database to events
```

```java
// ═══ DOMAIN TEST — pure, fast, no mocks ═══
class PaymentTest {

    @Test
    @DisplayName("Should transition from AUTHORIZED to CAPTURED")
    void shouldCapture() {
        var payment = Payment.initiate("cust-1", Money.of("100", "USD"),
            new PaymentMethod("card", "4242", "VISA"), "idem-1");
        payment.authorize("AUTH-001");

        payment.capture(Money.of("100", "USD"), "tx-001",
            new FeeBreakdown(new BigDecimal("2.90"), new BigDecimal("0.30"), new BigDecimal("3.20")));

        assertThat(payment.status()).isEqualTo(PaymentStatus.CAPTURED);
        assertThat(payment.domainEvents()).hasSize(3); // Initiated + Authorized + Captured
    }

    @Test
    @DisplayName("Should reject capture from INITIATED status")
    void shouldRejectCaptureFromWrongStatus() {
        var payment = Payment.initiate("cust-1", Money.of("100", "USD"),
            new PaymentMethod("card", "4242", "VISA"), "idem-1");

        assertThatThrownBy(() -> payment.capture(Money.of("100", "USD"), "tx-1",
                new FeeBreakdown(BigDecimal.ZERO, BigDecimal.ZERO, BigDecimal.ZERO)))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Cannot capture");
    }
}

// ═══ APPLICATION TEST — mocked ports ═══
class PaymentCommandServiceTest {

    @Mock private PaymentRepository paymentRepository;
    @Mock private PaymentGatewayPort gatewayPort;
    @Mock private FraudCheckPort fraudCheckPort;
    @Mock private PaymentEventPublisher eventPublisher;
    @Mock private NotificationPort notificationPort;

    private PaymentCommandService service;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        service = new PaymentCommandService(paymentRepository, gatewayPort,
            fraudCheckPort, eventPublisher, notificationPort);
    }

    @Test
    void shouldProcessPaymentSuccessfully() {
        when(paymentRepository.existsByIdempotencyKey(any())).thenReturn(false);
        when(fraudCheckPort.evaluate(any())).thenReturn(FraudResult.approved());
        when(gatewayPort.charge(any())).thenReturn(
            new ChargeResult("tx-1", "AUTH-1", "SUCCESS"));

        var command = new ProcessPaymentCommand("cust-1", Money.of("100", "USD"),
            PaymentType.CREDIT_CARD, new PaymentMethod("card", "4242", "VISA"),
            "idem-1", "corr-1");

        var result = service.process(command);

        assertThat(result.status()).isEqualTo(ResultStatus.SUCCESS);
        verify(paymentRepository).save(any());
        verify(eventPublisher, times(3)).publish(any()); // Initiated + Authorized + Captured
    }
}
```

---

## ✅ Pattern 6: Architecture Rules Enforcement (ArchUnit)

```java
@AnalyzeClasses(packages = "com.example.payment")
class HexagonalArchitectureTest {

    // ── Domain must not depend on application or infrastructure ──
    @ArchTest
    static final ArchRule domainIndependence =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..application..", "..infrastructure..");

    // ── Domain must not use Spring ──
    @ArchTest
    static final ArchRule domainNoFrameworks =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "org.springframework..",
                "org.apache.kafka..",
                "com.mongodb..",
                "org.hibernate..");

    // ── Application must not depend on infrastructure ──
    @ArchTest
    static final ArchRule applicationIndependence =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");

    // ── Adapters must only depend on ports, not on each other ──
    @ArchTest
    static final ArchRule adapterIsolation =
        noClasses().that().resideInAPackage("..adapter.in.rest..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..adapter.out..", "..adapter.in.kafka..", "..adapter.in.jms..");

    // ── Only adapters use Spring annotations ──
    @ArchTest
    static final ArchRule onlyAdaptersUseSpring =
        noClasses().that().resideInAPackage("..domain..")
            .should().beAnnotatedWith(Component.class)
            .orShould().beAnnotatedWith(Service.class)
            .orShould().beAnnotatedWith(Repository.class);

    // ── Ports are interfaces ──
    @ArchTest
    static final ArchRule portsAreInterfaces =
        classes().that().resideInAPackage("..port..")
            .should().beInterfaces();
}
```

---

## 🚫 Architecture Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Domain depends on Spring** | Can't test without framework, couples logic to infra | Zero framework imports in domain |
| **Anemic domain model** | All logic in services, entities are just data bags | Move behavior INTO domain entities |
| **No mapping between layers** | API change breaks domain, DB change breaks API | Separate DTOs per layer with mappers |
| **Skipping the port** | Controller calls repository directly | Always go through use case → port |
| **Business logic in adapters** | Validation in controller, rules in repository | Push all rules to domain/application |
| **God use case** | One service does everything | One use case per user intention |
| **Shared mutable state** | Domain entities with setters, shared across threads | Immutable value objects, state via methods |
| **Premature architecture** | Full hexagonal for simple CRUD | Scale architecture to complexity |

---

## ⚡ When to Use Which Architecture

| Scenario | Recommended | Why |
|----------|------------|-----|
| **Simple CRUD microservice** | Layered (Controller → Service → Repository) | Hexagonal is overkill |
| **Rich domain logic** (payments, insurance) | **Hexagonal** | Business rules deserve isolation |
| **Many external integrations** | **Hexagonal** | Ports isolate each integration |
| **Likely technology changes** | **Hexagonal** | Swap adapters without touching domain |
| **Multiple entry points** (REST + Kafka + JMS) | **Hexagonal** | Each entry point is a driving adapter |
| **Small team, fast iteration** | Package-by-feature | Less ceremony, faster delivery |
| **Large team, shared codebase** | Package-by-layer with ArchUnit | Enforced boundaries prevent coupling |

---

## 💡 Golden Rules of Clean/Hexagonal Architecture

```
1.  DEPENDENCIES POINT INWARD — domain never knows about infrastructure.
2.  DOMAIN IS KING — business logic lives in the domain layer, nowhere else.
3.  PORTS ARE INTERFACES — defined by the layer that NEEDS them (inner layer).
4.  ADAPTERS ARE REPLACEABLE — swap MongoDB for Postgres without touching domain.
5.  USE CASES ARE THIN — orchestrate domain objects and ports, don't contain business rules.
6.  MAP AT BOUNDARIES — each layer has its own data structures, mapped at transitions.
7.  TEST FROM THE INSIDE OUT — domain tests are fastest and most valuable.
8.  ENFORCE WITH ARCHUNIT — rules without enforcement are suggestions.
9.  SCALE TO COMPLEXITY — don't hexagonal a TODO app, don't layer-only a payment system.
10. PRAGMATISM OVER PURITY — the goal is maintainability, not architectural beauty contests.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / MongoDB / Kafka / Artemis / ArchUnit*
