# 🤖 GitHub Copilot Instructions for Enterprise Teams Cheat Sheet

> **Purpose:** Patterns for crafting effective Copilot instruction sets that enforce enterprise coding standards, architecture rules, and domain patterns.
> **Stack context:** Java 21+ / Spring Boot 3.x / GitHub Copilot / Enterprise teams

---

## 📋 Instruction Set Architecture

```
.github/
├── copilot-instructions.md          ← Global repo instructions
├── copilot/
│   ├── java-conventions.md          ← Language-specific
│   ├── spring-patterns.md           ← Framework-specific
│   ├── payment-domain.md            ← Domain-specific
│   └── testing-standards.md         ← Testing standards
```

---

## 📝 Pattern 1: Global Instruction File

### .github/copilot-instructions.md

```markdown
# Copilot Instructions — Payment Platform

## Tech Stack
- Java 21+ with preview features enabled
- Spring Boot 3.3.x
- MongoDB with Spring Data
- Apache Kafka with Spring Kafka
- Resilience4j for fault tolerance
- JUnit 5 + Testcontainers + AssertJ for testing
- Gradle 8.x with Kotlin DSL

## Architecture
- Hexagonal Architecture (Ports & Adapters)
- Domain layer: zero framework imports, pure business logic
- Application layer: use cases orchestrating domain + ports
- Infrastructure layer: Spring controllers, MongoDB repos, Kafka adapters

## Java Style
- Use records for value objects, DTOs, commands, events
- Use sealed interfaces for type hierarchies (events, commands, results)
- Use pattern matching switch expressions (exhaustive, no default on sealed types)
- Constructor injection only (no field @Autowired)
- Immutable by default (final fields, unmodifiable collections)
- Use ReentrantLock instead of synchronized (virtual thread compatibility)

## Naming
- Classes: PascalCase, descriptive nouns (FeeCalculator, not FeeHelper)
- Methods: camelCase, describe outcome (calculateFee, not process)
- Booleans: isX, hasX, canX, shouldX
- Constants: UPPER_SNAKE_CASE
- Packages: lowercase, singular (domain.model, not domain.models)

## Error Handling
- Never swallow exceptions (empty catch blocks)
- Specific exceptions, not catch(Exception e)
- Domain exceptions in domain layer, translated at boundaries
- Business rule violations return Result<T>, don't throw

## Do NOT
- Do not use field injection (@Autowired on fields)
- Do not use synchronized (use ReentrantLock)
- Do not use ThreadLocal (use ScopedValue or pass as parameter)
- Do not use Optional as a field, method parameter, or collection element
- Do not create mutable POJOs with getters/setters — use records
- Do not import Spring/framework classes in the domain layer
- Do not use parallelStream() — use virtual thread executor
- Do not write dual-writes (DB + Kafka) without outbox pattern
```

---

## 🏗️ Pattern 2: Domain-Specific Instructions

### .github/copilot/payment-domain.md

```markdown
# Payment Domain Instructions

## Domain Model
- Payment is the aggregate root with status state machine:
  INITIATED → AUTHORIZED → CAPTURED → (REFUNDED)
  INITIATED → FAILED (terminal)
- Money is a value object: record Money(BigDecimal value, String currency)
  - Always use BigDecimal for money, never double/float
  - Scale to 2 decimal places with RoundingMode.HALF_UP
  - Currency is ISO 4217 (3-letter code)

## State Transitions
- All status changes go through Payment.transitionTo() method
- Invalid transitions throw IllegalStateException
- Every transition records a StatusChange in the history

## Events
- Payment events use sealed interface:
  PaymentEvent.Initiated, Captured, Failed, Refunded
- Events include: paymentId, timestamp, and relevant data
- Events published via outbox pattern (never direct Kafka publish)

## Idempotency
- Every payment has an idempotencyKey (unique index in MongoDB)
- Check for existing idempotencyKey before processing
- Return cached result for duplicate idempotency keys

## Key Types
```java
// Use these patterns:
public sealed interface PaymentEvent permits
    PaymentEvent.Initiated, PaymentEvent.Captured,
    PaymentEvent.Failed, PaymentEvent.Refunded { ... }

public record Money(BigDecimal value, String currency) { ... }
public record FeeBreakdown(BigDecimal processing, BigDecimal platform, BigDecimal total) { ... }
```
```

---

## 🧪 Pattern 3: Testing Instructions

### .github/copilot/testing-standards.md

```markdown
# Testing Standards

## Test Naming
- Method: should[ExpectedBehavior]_when[Condition]
- Example: shouldRejectPayment_whenAmountIsNegative()
- Use @DisplayName for human-readable descriptions

## Test Structure
- AAA: Arrange, Act, Assert (one assert concept per test)
- Use AssertJ for all assertions (not JUnit assertEquals)
- Use @Nested classes for Given-When-Then grouping

## Test Types
- Unit tests: pure domain, no Spring, no mocks for domain services
- Application tests: mock all ports (interfaces), verify interactions
- Integration tests: Testcontainers for MongoDB/Kafka, @SpringBootTest
- Architecture tests: ArchUnit rules for dependency enforcement

## Frameworks
```java
// AssertJ style assertions (ALWAYS use this, not assertEquals)
assertThat(result.status()).isEqualTo(PaymentStatus.CAPTURED);
assertThat(result.amount().value()).isEqualByComparingTo("100.00");
assertThat(payments).hasSize(3).extracting(Payment::status)
    .containsExactly(CAPTURED, CAPTURED, FAILED);

// Mockito verification
verify(eventPublisher, times(1)).publish(any(PaymentEvent.Captured.class));
verify(gateway, never()).charge(any());

// Awaitility for async (NEVER Thread.sleep)
await().atMost(10, SECONDS).untilAsserted(() ->
    assertThat(consumer.getReceivedMessages()).hasSize(1));

// Testcontainers
@Container
static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0");
```

## Do NOT
- Do not use Thread.sleep() in tests — use Awaitility
- Do not test private methods directly
- Do not mock the class under test
- Do not write tests without meaningful assertions
- Do not use @SpringBootTest for unit tests (too slow)
```

---

## ⚙️ Pattern 4: Spring & Infrastructure Instructions

### .github/copilot/spring-patterns.md

```markdown
# Spring Patterns

## Configuration
- Use @ConfigurationProperties records (not @Value)
- Validate with @Validated and Bean Validation annotations
- Type-safe configuration with compact constructor defaults

```java
@ConfigurationProperties(prefix = "payment.gateway")
@Validated
public record GatewayProperties(
    @NotBlank String baseUrl,
    @Positive Duration connectTimeout,
    @Positive Duration readTimeout
) {
    public GatewayProperties {
        if (connectTimeout == null) connectTimeout = Duration.ofSeconds(3);
        if (readTimeout == null) readTimeout = Duration.ofSeconds(10);
    }
}
```

## REST Controllers
- Return ResponseEntity<T> with explicit status codes
- Use @Valid on request bodies
- Require Idempotency-Key header on POST endpoints
- Map to/from domain using dedicated mapper classes
- Global exception handler (@RestControllerAdvice) for consistent error format

## MongoDB
- Use MongoTemplate for complex queries, Spring Data repos for simple CRUD
- Atomic updates ($set, $inc, $push) over full document read-modify-write
- Always add indexes for query patterns (ESR rule)
- Use @Version for optimistic locking

## Kafka
- Manual acknowledgment (AckMode.MANUAL)
- Commit offsets AFTER processing, never before
- Include correlationId in Kafka headers
- Dead-letter topic for non-retryable errors
- Consumer must be idempotent

## Resilience
- Circuit breaker on all downstream service calls
- Retry only on transient, idempotent operations
- Timeout on every external call
- Fallback when non-critical enrichment fails
```

---

## 🎯 Pattern 5: Code Generation Prompts

### Effective Prompt Patterns for Copilot Chat

```
PATTERN: "Generate [WHAT] following [CONVENTION] for [CONTEXT]"

Examples:

"Generate a sealed interface for payment events with Initiated, Captured,
Failed, and Refunded subtypes as records. Each must include paymentId and
timestamp. Follow the pattern in PaymentEvent.java."

"Generate a MongoDB repository adapter implementing PaymentRepository port.
Use MongoTemplate with atomic updates. Follow hexagonal architecture —
this is an infrastructure adapter."

"Generate JUnit 5 tests for PaymentService.process() covering:
happy path, duplicate idempotency key, fraud rejection, gateway timeout.
Use AssertJ assertions and Mockito. Follow Given-When-Then with @Nested."

"Generate a Kafka consumer for payment.commands topic with manual
acknowledgment, error handling to DLT, and correlationId propagation.
Follow our Kafka consumer patterns."

"Generate a circuit breaker configuration for the payment gateway
with 50% failure threshold, 30s wait duration, and fallback. Use
Resilience4j with Spring Boot integration."
```

### Slash Commands for Team Workflow

```
/explain    → Explain this code focusing on domain logic and architecture decisions
/tests      → Generate tests following our testing-standards.md conventions
/fix        → Fix this code following our java-conventions and error handling standards
/doc        → Generate Javadoc for public API following our naming conventions
```

---

## 🔄 Pattern 6: Instruction Maintenance

```
KEEP INSTRUCTIONS:
  ✅ Concise (< 200 lines per file)
  ✅ Example-heavy (show patterns, not just rules)
  ✅ Updated with codebase (review quarterly)
  ✅ Team-reviewed (PRs for instruction changes)

AVOID:
  ❌ Novel-length instructions (Copilot context window is limited)
  ❌ Contradictory rules across files
  ❌ Rules without examples
  ❌ Stale patterns that don't match current code
```

---

## 💡 Golden Rules

```
1.  SHOW, don't just tell — code examples are 10x more effective than prose rules.
2.  DOMAIN INSTRUCTIONS are highest value — Copilot can't infer your business logic.
3.  NEGATIVE RULES prevent common mistakes — "Do NOT" lists catch AI autopilot errors.
4.  KEEP IT SHORT — Copilot has limited context, prioritize the most impactful rules.
5.  MATCH YOUR CODEBASE — instructions should reflect how your code actually looks.
6.  REVIEW QUARTERLY — stale instructions generate stale code.
7.  LAYER instructions — global → language → framework → domain → testing.
8.  TEST the instructions — periodically verify Copilot generates compliant code.
9.  TEAM OWNERSHIP — instructions are living docs, PRs required for changes.
10. COMPLEMENT, don't replace — instructions + code review + ArchUnit + linting = quality.
```

---

*Last updated: February 2026 | GitHub Copilot / Java 21+ / Spring Boot 3.x*
