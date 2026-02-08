# 🏗️ Before You Write Code — Software Design & Quality Cheat Sheet

> **Purpose:** A pre-coding checklist and reference guide. Consult this BEFORE writing any class, method, or module.
> **Author context:** Enterprise Java / Distributed Systems / Payment Processing

---

## 📋 Phase 0: The Pre-Coding Checklist

Before touching your IDE, answer these questions:

### 1. WHAT is the responsibility?
- [ ] Can I describe what this component does in **one sentence**?
- [ ] If I need "and" in that sentence, should this be **two components**?

### 2. WHO are the consumers?
- [ ] Who calls this? (API client, another service, Kafka consumer, scheduler)
- [ ] What contract do they expect? (sync/async, latency, error handling)

### 3. WHAT can go wrong?
- [ ] What are the failure modes? (timeout, null, invalid state, partial failure)
- [ ] How should each failure be handled? (retry, fallback, propagate, dead-letter)
- [ ] What's the blast radius if this fails?

### 4. HOW will I know it works?
- [ ] Can I write the test FIRST? (or at least sketch it)
- [ ] What are the edge cases?
- [ ] How will I observe it in production? (metrics, logs, traces)

### 5. WHAT will change?
- [ ] Which parts of this are likely to change? (business rules, integrations, data formats)
- [ ] Have I isolated the volatile parts behind abstractions?

---

## 🔷 S.O.L.I.D. Principles — Practical Guide

### S — Single Responsibility Principle (SRP)

> **"A class should have only one reason to change."**

**The Test:** If you describe a class and use "and", it likely violates SRP.

```
❌ PaymentService — validates payment AND calls gateway AND sends notification AND updates DB
✅ PaymentValidator, PaymentGatewayClient, PaymentNotifier, PaymentRepository
```

**Practical Heuristic:**
- If a class has more than **5-7 public methods**, question its cohesion
- If a class imports from **3+ unrelated domains**, it's doing too much
- If a **bug fix** in one behavior risks breaking another, split them

**Anti-pattern to watch:**
```
God Service → Controller calls one massive service that does everything
Fix → Decompose into focused collaborators orchestrated by a thin coordinator
```

---

### O — Open/Closed Principle (OCP)

> **"Open for extension, closed for modification."**

**The Test:** Can I add a new behavior WITHOUT editing existing code?

```java
// ❌ Closed for extension — must modify this switch for every new type
public BigDecimal calculateFee(PaymentType type, BigDecimal amount) {
    return switch (type) {
        case CREDIT_CARD -> amount.multiply(new BigDecimal("0.029"));
        case ACH -> new BigDecimal("0.50");
        // Must edit HERE to add WIRE, CRYPTO, etc.
    };
}

// ✅ Open for extension — just add a new implementation
public interface FeeCalculator {
    PaymentType supportedType();
    BigDecimal calculate(BigDecimal amount);
}

// Adding WIRE support = new class, zero changes to existing code
@Component
public class WireFeeCalculator implements FeeCalculator {
    public PaymentType supportedType() { return PaymentType.WIRE; }
    public BigDecimal calculate(BigDecimal amount) { return new BigDecimal("25.00"); }
}
```

**When to apply:** When you find yourself repeatedly modifying the same class/method to handle new variants.

**When NOT to over-apply:** Simple, stable logic (e.g., 3 status codes that won't change) doesn't need a strategy pattern. Pragmatism over dogma.

---

### L — Liskov Substitution Principle (LSP)

> **"Subtypes must be substitutable for their base types without breaking behavior."**

**The Test:** If I replace a parent with any child, does the program still behave correctly?

```java
// ❌ Violates LSP — Square breaks Rectangle's contract
class Rectangle {
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
}
class Square extends Rectangle {
    void setWidth(int w)  { this.width = w; this.height = w; } // Surprise!
}

// ❌ Violates LSP — throws where parent doesn't
class ReadOnlyRepository extends PaymentRepository {
    @Override
    public void save(Payment p) {
        throw new UnsupportedOperationException(); // Callers don't expect this
    }
}
```

**Practical Rules:**
- **Don't throw unexpected exceptions** in overridden methods
- **Don't strengthen preconditions** (accept less) or **weaken postconditions** (return less)
- **Don't silently ignore** operations the parent defines
- **Prefer composition over inheritance** when behaviors diverge

**Modern Java approach — use sealed types instead of deep hierarchies:**
```java
public sealed interface PaymentMethod
    permits CreditCard, BankAccount, DigitalWallet {}
// Each is self-contained, no inheritance surprises
```

---

### I — Interface Segregation Principle (ISP)

> **"No client should be forced to depend on methods it doesn't use."**

**The Test:** Does any implementor have methods that throw `UnsupportedOperationException` or do nothing?

```java
// ❌ Fat interface — forces implementors to stub methods they don't need
public interface MessageBroker {
    void publish(Message msg);
    Message consume();
    void acknowledge(String msgId);
    void setupDLQ(String queueName);
    HealthStatus healthCheck();
    Map<String, Long> getMetrics();
}

// ✅ Segregated — clients depend only on what they need
public interface MessagePublisher {
    void publish(Message msg);
}

public interface MessageConsumer {
    Message consume();
    void acknowledge(String msgId);
}

public interface Monitorable {
    HealthStatus healthCheck();
    Map<String, Long> getMetrics();
}
```

**Practical Heuristic:**
- If an interface has **more than 5 methods**, question if it should be split
- Group methods by **which clients actually use them together**
- Watch for interfaces that mix **commands** (write) with **queries** (read) — consider CQRS

---

### D — Dependency Inversion Principle (DIP)

> **"Depend on abstractions, not concretions."**

**The Test:** Can I swap the implementation without changing the caller?

```java
// ❌ High-level policy depends on low-level detail
public class PaymentService {
    private final StripeClient stripeClient;  // Concrete dependency
    private final PostgresRepository repo;    // Concrete dependency
}

// ✅ Depends on abstractions — swappable, testable
public class PaymentService {
    private final PaymentGateway gateway;      // Interface
    private final PaymentRepository repository; // Interface

    // Constructor injection — Spring wires the concrete implementations
    public PaymentService(PaymentGateway gateway, PaymentRepository repository) {
        this.gateway = gateway;
        this.repository = repository;
    }
}
```

**The Dependency Rule (Clean Architecture):**
```
Controller → Use Case (interface) → Entity
     ↓              ↓
  Framework    Gateway (interface)
  Adapter         ↓
              Infrastructure
              (DB, Queue, HTTP)

Dependencies point INWARD — inner layers never know about outer layers
```

**Practical Rules:**
- **Domain/business logic** should NEVER import framework classes
- **Constructor injection** over field injection — makes dependencies explicit and testable
- **Define interfaces in the domain layer**, implementations in the infrastructure layer

---

## 🎯 Beyond SOLID — Essential Design Principles

### DRY — Don't Repeat Yourself (But Don't Over-Abstract)

```
Rule of Three: Tolerate duplication twice. Abstract on the third occurrence.
Wrong DRY: Forcing unrelated code to share an abstraction just because it looks similar.
Right DRY: Extracting genuinely shared business logic into a single source of truth.
```

### YAGNI — You Aren't Gonna Need It

```
❌ "Let me add a plugin system in case we need it someday"
✅ Build what's needed NOW, design so it CAN be extended later (OCP)
```

### KISS — Keep It Simple

```
❌ AbstractStrategyFactoryProviderManagerImpl
✅ The simplest design that meets current requirements and is easy to change
```

### Tell, Don't Ask

```java
// ❌ Ask — pulls data out, makes decisions externally
if (account.getBalance().compareTo(payment.getAmount()) >= 0) {
    account.setBalance(account.getBalance().subtract(payment.getAmount()));
}

// ✅ Tell — object owns its logic
account.debit(payment.getAmount()); // throws InsufficientFundsException internally
```

### Law of Demeter (Don't talk to strangers)

```java
// ❌ Train wreck — reaching deep into object graphs
order.getCustomer().getAddress().getCity().getZipCode();

// ✅ Ask the immediate collaborator
order.shippingZipCode();
```

### Composition Over Inheritance

```java
// ❌ Fragile hierarchy
class EnhancedPaymentService extends PaymentService { ... }

// ✅ Compose behaviors
class PaymentService {
    private final FraudChecker fraudChecker;
    private final FeeCalculator feeCalculator;
    private final PaymentGateway gateway;
}
```

### Favor Immutability

```java
// ✅ Use records, final fields, unmodifiable collections
public record Payment(String id, Money amount, PaymentStatus status) {
    public Payment withStatus(PaymentStatus newStatus) {
        return new Payment(id, amount, newStatus);
    }
}
```

---

## 🏛️ Architectural Decision Checklist

### Before Creating a New Class

| Question | Action |
|----------|--------|
| Does a similar class already exist? | Extend or compose, don't duplicate |
| Can I describe it in one sentence without "and"? | If not, split into multiple classes |
| Does it depend on abstractions? | Inject interfaces, not concrete classes |
| Is it testable in isolation? | If not, extract dependencies |
| Are all fields final / immutable? | Default to immutable; mutability needs justification |

### Before Creating a New Method

| Question | Action |
|----------|--------|
| Does it do ONE thing? | If it has sections separated by blank lines, consider splitting |
| Is it < 15-20 lines? | Long methods hide complexity — extract sub-methods |
| Can I name it clearly without "and"? | `validateAndProcess()` → `validate()` + `process()` |
| Are all parameters necessary? | > 3 params → consider a parameter object (record) |
| Does it return a value OR cause a side effect? | Mixing both makes code unpredictable (CQS) |

### Before Creating an Interface

| Question | Action |
|----------|--------|
| Will there be more than one implementation? | No → skip the interface (YAGNI) |
| Is it for testability (mocking)? | Valid reason — create it |
| Does it hide a volatile dependency? (DB, API, queue) | Always abstract external dependencies |
| Does it have > 5 methods? | Consider splitting (ISP) |

---

## 🧪 Testing Mindset — Before Writing Code

### Test Categories to Plan For

```
Unit Tests        → Single class in isolation, mocked dependencies
                    Fast, deterministic, run on every build

Integration Tests → Component + real dependency (DB, queue, API)
                    Testcontainers for MongoDB, Kafka, Redis

Contract Tests    → API/event schema compatibility
                    Verify producer-consumer contracts

Edge Case Tests   → Nulls, empty collections, boundary values,
                    concurrent access, timeout scenarios
```

### Test-First Thinking

Before implementing, write (or sketch) tests for:
1. **Happy path** — does it work with valid input?
2. **Validation** — does it reject invalid input with clear errors?
3. **Failure handling** — does it behave correctly when dependencies fail?
4. **Concurrency** — is it safe under concurrent access?
5. **Idempotency** — does calling it twice produce the same result?

---

## 🛡️ Error Handling Strategy — Decide Before Coding

### Error Classification

| Type | Example | Strategy |
|------|---------|----------|
| **Recoverable / Transient** | Network timeout, DB connection lost | Retry with backoff |
| **Recoverable / Permanent** | Invalid input, business rule violation | Return error result, no retry |
| **Unrecoverable** | Config missing, corrupted state | Fail fast, alert, circuit break |
| **Partial Failure** | 3 of 5 batch items failed | Dead-letter failed items, continue |

### Error Handling Rules

```
1. NEVER swallow exceptions silently
2. NEVER use exceptions for control flow
3. ALWAYS translate infrastructure exceptions into domain errors
4. ALWAYS include context (what operation, what input, what state)
5. PREFER Result<T> / Either<Error, T> over throwing for expected failures
6. USE checked exceptions only at module boundaries (or not at all)
7. FAIL FAST on unrecoverable errors — don't propagate corrupted state
```

---

## 📐 Naming Conventions — Communicate Intent

### Classes
```
DO:    PaymentValidator, OrderRepository, KafkaPaymentPublisher
DON'T: PaymentHelper, OrderUtils, PaymentManager, DataHandler
       (these names reveal nothing about responsibility)
```

### Methods
```
DO:    calculateFee(), findByCustomerId(), publishPaymentEvent()
DON'T: process(), handle(), doStuff(), execute()
       (too vague — what does it process? handle what?)
```

### Boolean Methods
```
DO:    isValid(), hasBalance(), canProcess(), shouldRetry()
DON'T: checkValid(), validateStatus(), verify()
```

### Variables
```
DO:    remainingAttempts, settlementAmount, targetAccountId
DON'T: temp, data, result, flag, val, x
```

---

## 🔄 Design Patterns — When to Reach for Them

| Pattern | Reach For It When... |
|---------|---------------------|
| **Strategy** | Multiple algorithms/behaviors selected at runtime (fee calculation, retry policies) |
| **Builder** | Object has > 4 constructor params or optional fields |
| **Factory Method** | Creation logic is complex or varies by type |
| **Observer/Event** | Components need loose coupling (domain events, Kafka) |
| **Circuit Breaker** | Protecting against cascading failures in distributed calls |
| **Decorator** | Adding behavior (logging, metrics, retry) without modifying the original |
| **Template Method** | Shared algorithm skeleton with customizable steps |
| **Repository** | Abstracting data access behind a domain-oriented interface |
| **Specification** | Complex, composable business rules (`isEligible.and(isVerified)`) |

**Anti-patterns to avoid:**
- **Premature abstraction** — don't add patterns "just in case"
- **Pattern obsession** — simple `if/else` is fine for 2-3 cases
- **Anemic domain model** — don't put all logic in services with data-only entities

---

## 📏 Code Quality Metrics — Quick Gut Checks

| Metric | Healthy | Unhealthy |
|--------|---------|-----------|
| **Class size** | < 200 lines | > 500 lines |
| **Method size** | < 20 lines | > 40 lines |
| **Method parameters** | ≤ 3 | > 5 (use parameter object) |
| **Cyclomatic complexity** | < 10 per method | > 15 |
| **Import count** | < 10 unique domains | > 15 (doing too much) |
| **Test coverage** | > 80% line, > 70% branch | < 50% |
| **Dependency depth** | ≤ 3 layers | > 5 layers of wrapping |

---

## 🔁 The Pre-Commit Mental Checklist

Before pushing code, ask:

- [ ] **Does each class have a single, clear responsibility?**
- [ ] **Are dependencies injected, not created internally?**
- [ ] **Is the code testable in isolation?**
- [ ] **Have I handled ALL failure modes explicitly?**
- [ ] **Are there no magic numbers or unexplained strings?**
- [ ] **Would a new team member understand this in 5 minutes?**
- [ ] **Is there meaningful logging at entry, exit, and failure points?**
- [ ] **Have I considered thread safety?**
- [ ] **Are collections and objects immutable where possible?**
- [ ] **Does the public API reveal only what consumers need?**

---

## 💡 Golden Rules — The Non-Negotiables

```
1. Make it work → Make it right → Make it fast (in that order)
2. Code is read 10x more than it's written — optimize for readability
3. The best code is no code — solve the problem with less
4. Every abstraction has a cost — earn it with real use cases
5. If it's hard to test, the design is wrong
6. Explicit is better than implicit
7. Fail fast, fail loudly — silent failures are production incidents
8. Naming is design — if you can't name it clearly, you don't understand it
9. Comments explain WHY, code explains WHAT and HOW
10. Leave the codebase better than you found it
```

---

*Last updated: February 2026 | Optimized for Modern Java (17-24+) & Enterprise Distributed Systems*
