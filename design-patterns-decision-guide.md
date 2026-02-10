# 🧩 Design Patterns Decision Guide

> **Purpose:** When to use (and when NOT to use) each design pattern, with modern Java 21+ implementations. Reference before reaching for any pattern — the wrong pattern is worse than no pattern.
> **Stack context:** Java 21+ / Spring Boot 3.x / Records / Sealed Types / Pattern Matching

---

## 📋 The Pattern Decision Framework

Before applying any pattern, answer:

| Question | If YES | If NO |
|----------|--------|-------|
| Do I have **3+ concrete variants** of a behavior? | Strategy, Factory | Simple if/else is fine |
| Do I need to **compose behaviors** dynamically? | Decorator, Chain of Responsibility | Direct implementation |
| Is the **object construction** complex (>4 params, optional fields)? | Builder | Constructor or factory method |
| Do I need to **decouple** event producers from consumers? | Observer, Mediator | Direct call |
| Is there a **shared algorithm skeleton** with varying steps? | Template Method | Strategy (if steps are independent) |
| Do I need to **undo/track** state changes? | Memento, Command | Direct mutation |
| Am I **wrapping an incompatible** interface? | Adapter | Rewrite or extend |
| Do I need **one and only one** instance? | Singleton (enum) | Normal bean scope |

---

## 🏛️ Creational Patterns

### Factory Method

> **When:** Creation logic varies by type, or you want to decouple callers from concrete classes.

```java
// ── Modern Java: Sealed + Factory ──
public sealed interface PaymentProcessor permits
    CreditCardProcessor, AchProcessor, WireProcessor {

    PaymentResult process(Payment payment);

    // Factory method — returns the correct implementation
    static PaymentProcessor forType(PaymentType type) {
        return switch (type) {
            case CREDIT_CARD -> new CreditCardProcessor();
            case ACH         -> new AchProcessor();
            case WIRE        -> new WireProcessor();
        };
        // Compiler ensures exhaustiveness — no default needed
    }
}

// Usage
var processor = PaymentProcessor.forType(payment.type());
var result = processor.process(payment);
```

**Use when:** You have a type hierarchy and need to create the right subtype based on runtime data.
**Skip when:** Only 1-2 types that won't grow. A simple `new` is fine.

---

### Abstract Factory

> **When:** You need families of related objects that must be used together.

```java
// Payment gateway family — each gateway has its own charge, refund, and webhook handler
public sealed interface GatewayFactory permits StripeFactory, PayPalFactory, AdyenFactory {
    ChargeClient createChargeClient();
    RefundClient createRefundClient();
    WebhookParser createWebhookParser();

    static GatewayFactory forProvider(String provider) {
        return switch (provider.toLowerCase()) {
            case "stripe" -> new StripeFactory();
            case "paypal" -> new PayPalFactory();
            case "adyen"  -> new AdyenFactory();
            default -> throw new IllegalArgumentException("Unknown provider: " + provider);
        };
    }
}

// All Stripe components are consistent with each other
record StripeFactory() implements GatewayFactory {
    public ChargeClient createChargeClient() { return new StripeChargeClient(); }
    public RefundClient createRefundClient() { return new StripeRefundClient(); }
    public WebhookParser createWebhookParser() { return new StripeWebhookParser(); }
}
```

**Use when:** You have families of objects that MUST be used together (mixing Stripe charge + PayPal refund = bad).
**Skip when:** Objects aren't related or can be mixed freely.

---

### Builder

> **When:** Complex object construction with many optional parameters.

```java
// ── Modern Java: Records + Builder for immutable objects ──
public record PaymentRequest(
    String customerId,
    Money amount,
    PaymentType type,
    PaymentMethod method,
    String idempotencyKey,
    String description,
    Map<String, String> metadata,
    Address billingAddress,
    Instant scheduledFor
) {
    // Compact constructor for validation
    public PaymentRequest {
        Objects.requireNonNull(customerId, "customerId required");
        Objects.requireNonNull(amount, "amount required");
        Objects.requireNonNull(type, "type required");
        if (idempotencyKey == null) idempotencyKey = UUID.randomUUID().toString();
        if (metadata == null) metadata = Map.of();
    }

    public static Builder builder(String customerId, Money amount, PaymentType type) {
        return new Builder(customerId, amount, type);
    }

    public static class Builder {
        // Required
        private final String customerId;
        private final Money amount;
        private final PaymentType type;
        // Optional with defaults
        private PaymentMethod method;
        private String idempotencyKey;
        private String description;
        private Map<String, String> metadata = Map.of();
        private Address billingAddress;
        private Instant scheduledFor;

        private Builder(String customerId, Money amount, PaymentType type) {
            this.customerId = customerId;
            this.amount = amount;
            this.type = type;
        }

        public Builder method(PaymentMethod method) { this.method = method; return this; }
        public Builder idempotencyKey(String key) { this.idempotencyKey = key; return this; }
        public Builder description(String desc) { this.description = desc; return this; }
        public Builder metadata(Map<String, String> meta) { this.metadata = meta; return this; }
        public Builder billingAddress(Address addr) { this.billingAddress = addr; return this; }
        public Builder scheduledFor(Instant time) { this.scheduledFor = time; return this; }

        public PaymentRequest build() {
            return new PaymentRequest(customerId, amount, type, method,
                idempotencyKey, description, metadata, billingAddress, scheduledFor);
        }
    }
}

// Usage — clean, readable
var request = PaymentRequest.builder("cust-1", Money.of("100", "USD"), PaymentType.CREDIT_CARD)
    .method(new PaymentMethod("card", "4242", "VISA"))
    .description("Monthly subscription")
    .metadata(Map.of("subscription_id", "sub-123"))
    .build();
```

**Use when:** >4 constructor params, optional fields, or step-by-step construction.
**Skip when:** ≤3 required fields and no optional fields — use constructor or record directly.

---

### Singleton

> **When:** Exactly one instance needed globally.

```java
// ── Modern Java: Enum singleton (Effective Java Item 3) ──
public enum ApplicationConfig {
    INSTANCE;

    private final Map<String, String> properties = new ConcurrentHashMap<>();

    public void load(Map<String, String> props) { properties.putAll(props); }
    public String get(String key) { return properties.get(key); }
    public String get(String key, String defaultValue) {
        return properties.getOrDefault(key, defaultValue);
    }
}

// In Spring Boot: just use @Bean (singleton scope by default)
// Don't implement singleton pattern manually — Spring handles it
@Bean // Singleton by default
public ObjectMapper objectMapper() {
    return new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
}
```

**Use when:** Configuration holders, caches, connection pools (outside Spring context).
**Skip when:** Using Spring — all beans are singleton by default. Don't reinvent it.

---

## 🏗️ Structural Patterns

### Adapter

> **When:** Making an incompatible interface work with your existing code.

```java
// External Stripe SDK has its own types — adapt to our port interface
public class StripeGatewayAdapter implements PaymentGatewayPort {

    private final StripeClient stripeClient;    // External SDK

    @Override
    public ChargeResult charge(ChargeRequest request) {
        // Adapt OUR domain types → Stripe SDK types
        var stripeCharge = StripeChargeCreateParams.builder()
            .setAmount(request.amount().toCents())
            .setCurrency(request.currency().toLowerCase())
            .setSource(request.paymentMethodToken())
            .setIdempotencyKey(request.idempotencyKey())
            .build();

        // Call Stripe
        var stripeResponse = stripeClient.charges().create(stripeCharge);

        // Adapt Stripe response → OUR domain types
        return new ChargeResult(
            stripeResponse.getId(),
            stripeResponse.getAuthorizationCode(),
            mapStripeStatus(stripeResponse.getStatus())
        );
    }
}
```

**Use when:** Integrating third-party libraries, legacy systems, or external APIs into your port interfaces.
**Skip when:** You control both sides — just align the interfaces.

---

### Decorator

> **When:** Adding behavior to an object without modifying it. Behaviors can be composed.

```java
// Base interface
public interface PaymentProcessor {
    PaymentResult process(Payment payment);
}

// Core implementation
public class CorePaymentProcessor implements PaymentProcessor {
    public PaymentResult process(Payment payment) {
        return gateway.charge(payment);
    }
}

// Decorators — each adds one concern
public class LoggingDecorator implements PaymentProcessor {
    private final PaymentProcessor delegate;

    public PaymentResult process(Payment payment) {
        log.info("Processing payment: {}", payment.id());
        var result = delegate.process(payment);
        log.info("Payment result: {} → {}", payment.id(), result.status());
        return result;
    }
}

public class MetricsDecorator implements PaymentProcessor {
    private final PaymentProcessor delegate;
    private final MeterRegistry registry;

    public PaymentResult process(Payment payment) {
        return registry.timer("payment.process.duration", "type", payment.type().name())
            .record(() -> delegate.process(payment));
    }
}

public class RetryDecorator implements PaymentProcessor {
    private final PaymentProcessor delegate;
    private final Retry retry;

    public PaymentResult process(Payment payment) {
        return Retry.decorateSupplier(retry, () -> delegate.process(payment)).get();
    }
}

// Compose — each layer wraps the previous
PaymentProcessor processor = new RetryDecorator(
    new MetricsDecorator(
        new LoggingDecorator(
            new CorePaymentProcessor(gateway)
        ), registry
    ), retry
);

// In Spring — use @Primary and @Qualifier, or manual bean wiring
@Configuration
public class ProcessorConfig {
    @Bean
    public PaymentProcessor paymentProcessor(
            PaymentGateway gateway, MeterRegistry registry, Retry retry) {
        return new RetryDecorator(
            new MetricsDecorator(
                new LoggingDecorator(
                    new CorePaymentProcessor(gateway)
                ), registry
            ), retry
        );
    }
}
```

**Use when:** Cross-cutting concerns (logging, metrics, retry, caching, auth) that can be mixed and matched.
**Skip when:** Only one decorator and it won't change — just put the logic inline.

---

### Facade

> **When:** Simplifying a complex subsystem behind a clean interface.

```java
// Complex subsystem: multiple services needed for payment processing
public class PaymentFacade {

    private final AccountService accountService;
    private final FraudService fraudService;
    private final GatewayService gatewayService;
    private final FeeCalculator feeCalculator;
    private final NotificationService notifications;

    // One simple method hides the complex orchestration
    public PaymentResult processPayment(PaymentRequest request) {
        accountService.validateAccount(request.accountId());
        fraudService.screen(request);
        var fees = feeCalculator.calculate(request);
        var result = gatewayService.charge(request, fees);
        notifications.sendConfirmation(request, result);
        return result;
    }
}
```

**Use when:** External consumers need a simplified API to a complex internal system.
**Skip when:** The subsystem is already simple — a facade over one class is pointless.

---

## 🎭 Behavioral Patterns

### Strategy

> **When:** Multiple algorithms for the same task, selected at runtime.

```java
// ── Modern Java: Enum + interface strategy ──
public interface FeeStrategy {
    Money calculateFee(Money amount);
}

public enum PaymentType implements FeeStrategy {
    CREDIT_CARD {
        public Money calculateFee(Money amount) {
            return amount.multiply(new BigDecimal("0.029"));
        }
    },
    ACH {
        public Money calculateFee(Money amount) {
            return Money.of("0.50", amount.currency());
        }
    },
    WIRE {
        public Money calculateFee(Money amount) {
            return Money.of("25.00", amount.currency());
        }
    };
}

// ── Spring-managed strategies (when strategies need injected dependencies) ──
public interface NotificationStrategy {
    NotificationChannel channel();
    void send(String recipient, String message);
}

@Component
public class EmailNotification implements NotificationStrategy {
    private final EmailClient emailClient;  // Spring-injected
    public NotificationChannel channel() { return NotificationChannel.EMAIL; }
    public void send(String recipient, String message) { emailClient.send(recipient, message); }
}

@Component
public class SmsNotification implements NotificationStrategy {
    private final SmsGateway smsGateway;  // Spring-injected
    public NotificationChannel channel() { return NotificationChannel.SMS; }
    public void send(String recipient, String message) { smsGateway.send(recipient, message); }
}

// Registry pattern — auto-discovers strategies via Spring DI
@Component
public class NotificationRouter {
    private final EnumMap<NotificationChannel, NotificationStrategy> strategies;

    public NotificationRouter(List<NotificationStrategy> allStrategies) {
        this.strategies = new EnumMap<>(NotificationChannel.class);
        allStrategies.forEach(s -> strategies.put(s.channel(), s));
    }

    public void notify(NotificationChannel channel, String recipient, String message) {
        strategies.get(channel).send(recipient, message);
    }
}
```

**Use when:** 3+ interchangeable algorithms, especially when they need different dependencies.
**Skip when:** 2 simple variants — a switch expression or if/else is clearer.

---

### Template Method

> **When:** Shared algorithm skeleton with customizable steps.

```java
// ── Abstract class defines the skeleton ──
public abstract class PaymentProcessingTemplate {

    // Template method — defines the fixed sequence
    public final PaymentResult process(Payment payment) {
        validate(payment);
        var enriched = enrich(payment);
        var fees = calculateFees(enriched);
        var result = executeCharge(enriched, fees);
        postProcess(enriched, result);
        return result;
    }

    // Fixed steps (can be overridden but usually aren't)
    protected void validate(Payment payment) {
        if (payment.amount().isNegativeOrZero()) {
            throw new ValidationException("Amount must be positive");
        }
    }

    // Steps that MUST be customized
    protected abstract Payment enrich(Payment payment);
    protected abstract FeeBreakdown calculateFees(Payment payment);
    protected abstract PaymentResult executeCharge(Payment payment, FeeBreakdown fees);

    // Hook — optional override
    protected void postProcess(Payment payment, PaymentResult result) {
        // Default: do nothing. Subclasses can override.
    }
}

// Concrete: credit card flow
public class CreditCardProcessing extends PaymentProcessingTemplate {
    protected Payment enrich(Payment payment) {
        return payment.withBinLookup(binService.lookup(payment.cardNumber()));
    }
    protected FeeBreakdown calculateFees(Payment payment) {
        return new FeeBreakdown(payment.amount().multiply("0.029"), Money.of("0.30", "USD"));
    }
    protected PaymentResult executeCharge(Payment payment, FeeBreakdown fees) {
        return stripeGateway.charge(payment, fees);
    }
}

// Concrete: ACH flow
public class AchProcessing extends PaymentProcessingTemplate {
    protected Payment enrich(Payment payment) {
        return payment.withBankVerification(achService.verify(payment.routingNumber()));
    }
    protected FeeBreakdown calculateFees(Payment payment) {
        return new FeeBreakdown(Money.of("0.50", "USD"), Money.ZERO);
    }
    protected PaymentResult executeCharge(Payment payment, FeeBreakdown fees) {
        return achGateway.initiate(payment);  // ACH is async
    }
    protected void postProcess(Payment payment, PaymentResult result) {
        webhookRegistrar.registerForSettlement(payment.id());  // ACH-specific
    }
}
```

**Use when:** Multiple variants share 70%+ of the same algorithm, with a few varying steps.
**Skip when:** Variants differ significantly — use Strategy instead (composition over inheritance).

---

### Chain of Responsibility

> **When:** A request needs to pass through a series of handlers, each deciding whether to process or pass along.

```java
// ── Modern Java: functional chain ──
public interface PaymentValidationRule {
    Optional<ValidationError> validate(Payment payment);
}

@Component public class AmountLimitRule implements PaymentValidationRule {
    public Optional<ValidationError> validate(Payment payment) {
        if (payment.amount().value().compareTo(new BigDecimal("50000")) > 0) {
            return Optional.of(new ValidationError("AMOUNT_LIMIT", "Exceeds maximum"));
        }
        return Optional.empty();
    }
}

@Component public class SanctionsRule implements PaymentValidationRule {
    public Optional<ValidationError> validate(Payment payment) {
        if (sanctionsList.contains(payment.customerId())) {
            return Optional.of(new ValidationError("SANCTIONED", "Customer is sanctioned"));
        }
        return Optional.empty();
    }
}

@Component public class VelocityRule implements PaymentValidationRule {
    public Optional<ValidationError> validate(Payment payment) {
        long recentCount = repository.countRecentPayments(payment.customerId(), Duration.ofHours(1));
        if (recentCount > 10) {
            return Optional.of(new ValidationError("VELOCITY", "Too many payments in 1 hour"));
        }
        return Optional.empty();
    }
}

// Chain runner — collects ALL errors (doesn't short-circuit)
@Component
public class PaymentValidationChain {
    private final List<PaymentValidationRule> rules;  // Spring auto-injects all

    public ValidationResult validate(Payment payment) {
        var errors = rules.stream()
            .map(rule -> rule.validate(payment))
            .filter(Optional::isPresent)
            .map(Optional::get)
            .toList();
        return errors.isEmpty() ? ValidationResult.success() : ValidationResult.failure(errors);
    }
}
```

**Use when:** Multiple independent validators/filters/handlers that can be composed.
**Skip when:** Fixed sequence of steps that always all run — just call them sequentially.

---

### Observer / Event

> **When:** Decoupling event producers from consumers. Multiple reactions to one event.

```java
// ── Spring Application Events (in-process) ──
// Domain event
public record PaymentCompletedEvent(String paymentId, Money amount, String customerId) {}

// Publisher
@Component
public class PaymentService {
    private final ApplicationEventPublisher eventPublisher;

    public void completePayment(Payment payment) {
        // ... business logic ...
        eventPublisher.publishEvent(new PaymentCompletedEvent(
            payment.id(), payment.amount(), payment.customerId()));
    }
}

// Listeners — decoupled, independently deployable
@Component
public class SendReceiptListener {
    @EventListener
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        emailService.sendReceipt(event.customerId(), event.amount());
    }
}

@Component
public class UpdateAnalyticsListener {
    @EventListener
    @Async  // Non-blocking
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        analyticsService.recordPayment(event);
    }
}

@Component
public class TriggerLoyaltyListener {
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT) // Only after TX commits
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        loyaltyService.addPoints(event.customerId(), event.amount());
    }
}
```

**Use when:** Multiple independent reactions to an event; callers shouldn't know about listeners.
**Skip when:** Only one listener and it's tightly coupled — direct call is simpler.

---

### Command

> **When:** Encapsulate a request as an object for queuing, logging, undoing, or replaying.

```java
// ── Commands as sealed records ──
public sealed interface PaymentCommand {
    String paymentId();

    record Capture(String paymentId, Money amount) implements PaymentCommand {}
    record Refund(String paymentId, Money amount, String reason) implements PaymentCommand {}
    record Cancel(String paymentId, String reason) implements PaymentCommand {}
    record Retry(String paymentId, int attemptNumber) implements PaymentCommand {}
}

// Command handler with exhaustive pattern matching
@Component
public class PaymentCommandHandler {

    public PaymentResult handle(PaymentCommand command) {
        return switch (command) {
            case PaymentCommand.Capture c  -> capturePayment(c);
            case PaymentCommand.Refund r   -> refundPayment(r);
            case PaymentCommand.Cancel c   -> cancelPayment(c);
            case PaymentCommand.Retry r    -> retryPayment(r);
        };
    }
}

// Commands can be queued, logged, replayed
@KafkaListener(topics = "payment.commands")
public void onCommand(PaymentCommand command, Acknowledgment ack) {
    commandHandler.handle(command);
    ack.acknowledge();
}
```

**Use when:** You need to queue, serialize, log, undo, or replay operations.
**Skip when:** Direct method calls work fine and you don't need the indirection.

---

### Specification

> **When:** Complex, composable business rules that need to be combined dynamically.

```java
// ── Functional specification pattern ──
@FunctionalInterface
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }
}

// Concrete specifications
public class PaymentSpecs {
    public static Specification<Payment> isHighValue() {
        return p -> p.amount().value().compareTo(new BigDecimal("10000")) >= 0;
    }
    public static Specification<Payment> isCreditCard() {
        return p -> p.type() == PaymentType.CREDIT_CARD;
    }
    public static Specification<Payment> isNewCustomer() {
        return p -> p.customerTenureDays() < 30;
    }
    public static Specification<Payment> requiresManualReview() {
        return isHighValue().and(isNewCustomer());
    }
    public static Specification<Payment> requiresExtraFraudCheck() {
        return isHighValue().and(isCreditCard()).or(isNewCustomer());
    }
}

// Usage — composable, readable
if (PaymentSpecs.requiresManualReview().isSatisfiedBy(payment)) {
    routeToManualQueue(payment);
}

var eligible = payments.stream()
    .filter(PaymentSpecs.isHighValue().and(PaymentSpecs.isCreditCard())::isSatisfiedBy)
    .toList();
```

**Use when:** Business rules that combine in multiple ways across different contexts.
**Skip when:** Simple boolean checks that don't compose.

---

## 📐 Pattern Selection Quick Reference

| Scenario | Pattern | Modern Java Approach |
|----------|---------|---------------------|
| Multiple algorithms for same task | **Strategy** | Enum implementing interface, or Spring DI registry |
| Complex object construction | **Builder** | Record + nested Builder class |
| Shared algorithm, varying steps | **Template Method** | Abstract class (or Strategy if preferring composition) |
| Wrap incompatible interface | **Adapter** | Port implementation delegating to external SDK |
| Add behavior without modifying | **Decorator** | Wrapper implementing same interface, composable |
| Multiple validators/filters | **Chain of Responsibility** | `List<Rule>` with Spring auto-injection |
| Decouple event producer/consumer | **Observer** | Spring `@EventListener` or Kafka events |
| Encapsulate request as object | **Command** | Sealed interface + record subtypes |
| Complex composable business rules | **Specification** | Functional interface with and/or/not combinators |
| Create correct subtype by type | **Factory Method** | Static factory on sealed interface |
| Simplify complex subsystem | **Facade** | Thin coordinator class |
| Exactly one instance | **Singleton** | Enum singleton or Spring `@Bean` |
| Family of related objects | **Abstract Factory** | Sealed interface per family |

---

## 🚫 Pattern Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Pattern for 2 cases** | Over-engineering simple logic | Use if/else until 3rd case arrives |
| **Strategy without polymorphism** | Just a fancy switch | Switch expression is clearer for simple cases |
| **Singleton for everything** | Global state, hard to test | Use Spring DI, scope beans properly |
| **Decorator for one concern** | Unnecessary indirection | Put logic inline |
| **Abstract Factory with one product** | Over-abstraction | Simple Factory Method |
| **Observer with one listener** | Decoupling nothing | Direct method call |
| **Builder for 2-field object** | Ceremony for no benefit | Use constructor or record |
| **Template Method with deep hierarchies** | Fragile inheritance chains | Prefer Strategy (composition) |
| **Pattern obsession** | Every class has a pattern label | Use patterns to solve problems, not to look smart |

---

## 💡 Golden Rules of Design Patterns

```
1.  PATTERNS SOLVE PROBLEMS — if you can't name the problem, don't apply the pattern.
2.  RULE OF THREE — tolerate duplication twice, abstract on the third occurrence.
3.  COMPOSITION OVER INHERITANCE — prefer Strategy (interface) over Template Method (abstract class).
4.  MODERN JAVA CHANGES EVERYTHING — sealed types, records, and pattern matching replace many GoF patterns.
5.  SPRING IS YOUR FACTORY — DI eliminates manual Factory/Singleton in most cases.
6.  ENUMS ARE STRATEGIES — enum implementing interface is the cleanest strategy in Java.
7.  SWITCH EXPRESSIONS are pattern matching — they replace Visitor and simplify Command handlers.
8.  If a pattern makes code HARDER to understand, you're doing it wrong.
9.  The best code often uses NO named pattern — it just follows SOLID principles naturally.
10. KNOW the patterns so you RECOGNIZE them, not so you FORCE them everywhere.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Records / Sealed Types / Pattern Matching*
