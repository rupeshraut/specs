# 🔍 Code Review Checklist

> **Purpose:** Systematic checklist for reviewing enterprise Java code. Use as a reviewer to catch issues, or as an author for self-review before submitting a PR.
> **Stack context:** Java 21+ / Spring Boot 3.x / Kafka / MongoDB / Resilience4j

---

## 📋 The Review Mindset

```
Priority order when reviewing:
  1. CORRECTNESS   — Does it do what it should? Are edge cases handled?
  2. SECURITY      — Can it be exploited? Does it leak data?
  3. RELIABILITY   — What happens when things fail? Is it resilient?
  4. MAINTAINABILITY — Can someone else understand and modify this?
  5. PERFORMANCE   — Is it efficient enough? (Don't optimize prematurely)
  6. STYLE         — Naming, formatting, conventions (least important)
```

---

## ✅ Section 1: Correctness

### Logic & Behavior

```
[ ] Does the code handle ALL input states? (null, empty, boundary values)
[ ] Are all conditional branches reachable and tested?
[ ] Are comparison operators correct? (< vs <=, == vs .equals())
[ ] Are BigDecimal comparisons using compareTo(), NOT equals()?
      ❌ amount.equals(BigDecimal.ZERO)     // "0.00" != "0" in equals!
      ✅ amount.compareTo(BigDecimal.ZERO) == 0
[ ] Are floating point values avoided for money? (Use BigDecimal)
[ ] Is integer overflow possible? (int max = 2.1B)
[ ] Are off-by-one errors avoided in loops and ranges?
[ ] Are switch expressions exhaustive? (sealed types help)
[ ] Is the happy path AND the sad path implemented?
```

### Null Safety

```
[ ] Are null checks present where needed?
[ ] Is Optional used for return types that might be absent?
[ ] Is Optional.get() never called without isPresent() check?
      ❌ repository.findById(id).get()
      ✅ repository.findById(id).orElseThrow(() -> new NotFoundException(id))
[ ] Are @NonNull/@Nullable annotations used on public APIs?
[ ] Are record compact constructors rejecting nulls?
      public PaymentAmount {
          Objects.requireNonNull(value, "value must not be null");
      }
```

### Concurrency

```
[ ] Is shared mutable state protected? (lock, atomic, or eliminated)
[ ] Are collections thread-safe if accessed concurrently?
      ❌ new HashMap<>() shared across threads
      ✅ new ConcurrentHashMap<>() or Collections.unmodifiableMap()
[ ] Are lazy initialization patterns thread-safe?
[ ] Is synchronized avoided with virtual threads? (Use ReentrantLock)
[ ] Are CompletableFuture chains properly handling exceptions?
[ ] Is there a race condition between check-then-act?
      ❌ if (!exists(key)) { insert(key); }   // TOCTOU race
      ✅ Use atomic putIfAbsent or unique index constraint
```

---

## 🔒 Section 2: Security

### Input Validation

```
[ ] Is ALL external input validated before use?
[ ] Are Bean Validation annotations present on DTOs? (@NotNull, @Positive, @Size)
[ ] Is input size bounded? (prevent DoS via large payloads)
[ ] Are path/query parameters validated? (@PathVariable with pattern)
[ ] Are SQL/NoSQL injection vectors prevented?
      ❌ mongoTemplate.find(Query.query(Criteria.where("name").is(userInput)))
         when userInput could be a MongoDB operator like {"$gt": ""}
      ✅ Validate/sanitize input, use parameterized queries
```

### Data Protection

```
[ ] Are secrets NEVER hardcoded? (passwords, API keys, tokens)
[ ] Is PII masked in logs? (email, phone, card numbers)
      ❌ log.info("Processing payment for card: {}", cardNumber);
      ✅ log.info("Processing payment for card: ****{}", last4Digits);
[ ] Are sensitive fields excluded from toString()?
[ ] Are error messages generic to clients? (don't leak stack traces)
      ❌ return ResponseEntity.status(500).body(exception.getMessage());
      ✅ return ResponseEntity.status(500).body("Internal server error");
[ ] Is authorization checked? (not just authentication)
[ ] Are API responses free of internal implementation details?
```

### Dependencies

```
[ ] Are dependencies from trusted sources?
[ ] Are known CVEs checked? (mvn dependency-check:check)
[ ] Are dependency versions pinned? (no LATEST or SNAPSHOT in prod)
[ ] Is the principle of least privilege applied to service accounts?
```

---

## 🛡️ Section 3: Reliability & Resilience

### Error Handling

```
[ ] Are exceptions NOT swallowed silently?
      ❌ try { ... } catch (Exception e) { }              // Silent swallow
      ❌ try { ... } catch (Exception e) { e.printStackTrace(); }  // Not production
      ✅ try { ... } catch (Exception e) { log.error("Context: {}", context, e); throw ...; }
[ ] Are specific exceptions caught (not just Exception)?
[ ] Are checked exceptions translated at layer boundaries?
      ❌ throws SQLException from service layer
      ✅ catch SQLException, throw DomainException
[ ] Is exception context preserved when re-throwing?
      ❌ throw new ServiceException(e.getMessage());     // Loses stack trace
      ✅ throw new ServiceException("Context", e);       // Preserves cause chain
[ ] Are validation errors returned, not thrown? (for expected failures)
      ✅ return ValidationResult.failure(errors);
[ ] Does error handling distinguish transient vs permanent failures?
```

### External Calls

```
[ ] Does EVERY external call have a timeout?
      ❌ restTemplate.getForObject(url, Response.class);  // No timeout
[ ] Are circuit breakers configured for downstream services?
[ ] Is retry logic present for transient failures?
[ ] Is retry ONLY on idempotent operations?
[ ] Are retries classified correctly? (never retry 400, always retry 503)
[ ] Is there a fallback for when the downstream is unavailable?
[ ] Are connection pools bounded?
[ ] Is backpressure handled for async operations?
```

### Data Integrity

```
[ ] Are operations idempotent where needed? (Kafka consumers, API retries)
[ ] Is the outbox pattern used for DB + event publishing?
      ❌ repository.save(entity); kafkaTemplate.send(event);  // Dual write!
[ ] Are database constraints enforced? (unique indexes, foreign keys)
[ ] Is optimistic locking used for concurrent updates?
      ✅ @Version Long version; on the document
[ ] Are partial failures handled in batch operations?
[ ] Is data validated at system boundaries (API input, Kafka deserialization)?
```

---

## 🏗️ Section 4: Design & Maintainability

### SOLID Principles

```
[ ] SRP: Does each class have ONE reason to change?
      ❌ PaymentService that validates, charges, notifies, and logs
      ✅ PaymentValidator, PaymentGatewayClient, NotificationService
[ ] OCP: Can new behavior be added without modifying existing code?
      ❌ Adding new payment type requires editing a switch in 5 places
      ✅ New payment type = new Strategy implementation
[ ] LSP: Can subtypes replace parent types without breaking behavior?
      ❌ ReadOnlyRepo extends Repo { save() { throw UnsupportedOp(); } }
[ ] ISP: Do interfaces have only methods their clients need?
      ❌ Interface with 12 methods, implementors stub half of them
[ ] DIP: Does the code depend on abstractions, not concretions?
      ❌ private final StripeClient client;      // Concrete
      ✅ private final PaymentGatewayPort gateway; // Interface
```

### Code Structure

```
[ ] Are classes < 200 lines? (if not, should they be split?)
[ ] Are methods < 20 lines? (if not, should they be extracted?)
[ ] Are methods doing ONE thing? (no validateAndProcess)
[ ] Are method parameters ≤ 3? (use parameter object if more)
[ ] Is the method return type clear? (avoid returning null for "not found")
[ ] Is there no dead code? (unused methods, commented-out code, unreachable branches)
[ ] Are magic numbers and strings replaced with named constants?
      ❌ if (retryCount > 3)
      ✅ if (retryCount > MAX_RETRY_ATTEMPTS)
[ ] Are utility methods in the right place? (not duplicated across classes)
```

### Naming

```
[ ] Do class names describe WHAT, not HOW?
      ❌ PaymentHelper, DataProcessor, Utils
      ✅ FeeCalculator, PaymentValidator, IdempotencyGuard
[ ] Do method names describe the OUTCOME?
      ❌ process(), handle(), doStuff()
      ✅ calculateFee(), findByCustomerId(), publishPaymentEvent()
[ ] Are boolean methods named as questions?
      ✅ isValid(), hasBalance(), canProcess(), shouldRetry()
[ ] Are variables named for their MEANING, not their type?
      ❌ String str, int num, List list
      ✅ String customerId, int remainingAttempts, List<Payment> pendingPayments
[ ] Are abbreviations avoided? (use customerId, not custId)
```

### Immutability

```
[ ] Are value objects implemented as records?
[ ] Are all record/class fields final?
[ ] Are collections returned as unmodifiable?
      ❌ return this.items;                    // Caller can mutate internal list
      ✅ return Collections.unmodifiableList(items);
      ✅ return List.copyOf(items);
[ ] Are "with" methods used instead of setters?
      ✅ payment.withStatus(CAPTURED)  // Returns new instance
[ ] Is mutable state justified? (default should be immutable)
```

---

## 🗄️ Section 5: Data & Database

### MongoDB Specific

```
[ ] Are indexes defined for all query patterns?
[ ] Does the index follow ESR rule? (Equality → Sort → Range)
[ ] Are projections used? (don't load fields you don't need)
      ❌ mongoTemplate.find(query, Payment.class);  // Loads entire document
      ✅ query.fields().include("id", "status", "amount");
[ ] Is auto-index-creation disabled in production?
[ ] Are aggregation pipelines using $match early? (filter before group)
[ ] Are embedded arrays bounded? (won't grow past 16MB doc limit)
[ ] Are atomic operations used where possible?
      ❌ Read → modify in code → save entire document
      ✅ $set, $inc, $push for field-level updates
[ ] Is TTL configured for ephemeral data? (events, sessions, locks)
```

### Kafka Specific

```
[ ] Is the message key chosen for correct partitioning/ordering?
[ ] Is the consumer idempotent?
[ ] Are offsets committed AFTER processing, not before?
      ❌ ack.acknowledge(); process(event);  // Message lost on crash
      ✅ process(event); ack.acknowledge();
[ ] Are non-retryable exceptions excluded from retry?
[ ] Is there a dead-letter topic configured?
[ ] Are Kafka headers propagating correlationId?
[ ] Is the schema backward/forward compatible?
[ ] Is acks=all for critical topics?
[ ] Is max.poll.interval.ms sufficient for processing time?
```

---

## ⚡ Section 6: Performance

### General

```
[ ] Are N+1 query patterns avoided?
      ❌ customers.forEach(c -> repo.findPayments(c.id()));  // N queries
      ✅ repo.findPaymentsByCustomerIds(customerIds);         // 1 query
[ ] Is pagination used for large result sets?
      ❌ repo.findAll();                    // Loads everything into memory
      ✅ repo.findAll(PageRequest.of(0, 100));
[ ] Are expensive objects created once and reused?
      ❌ new ObjectMapper() inside a loop
      ✅ static final ObjectMapper MAPPER = ...
[ ] Is string concatenation in loops using StringBuilder?
[ ] Are streams used appropriately? (not for simple 1-element operations)
[ ] Is parallelStream() avoided? (prefer virtual threads or explicit executor)
[ ] Are there unnecessary copies of large collections?
```

### Spring Specific

```
[ ] Is constructor injection used? (not field @Autowired)
      ❌ @Autowired private PaymentService service;
      ✅ private final PaymentService service; // via constructor
[ ] Are beans in the right scope? (singleton for stateless, prototype for stateful)
[ ] Are database connections returned promptly? (no long-held connections)
[ ] Is @Async used with proper executor configuration?
[ ] Are @Transactional boundaries as narrow as possible?
[ ] Is lazy loading of unnecessary beans considered for startup time?
```

---

## 📝 Section 7: Testing

```
[ ] Are there tests? (obvious but: no tests = no approval)
[ ] Do tests cover happy path AND failure paths?
[ ] Do tests have meaningful names?
      ❌ test1(), testProcess(), testPayment()
      ✅ shouldRejectPayment_whenAmountIsNegative()
[ ] Does each test have at least one meaningful assertion?
      ❌ Test that calls a method but asserts nothing
[ ] Are mocks used appropriately? (mock dependencies, not the SUT)
[ ] Are integration tests using Testcontainers? (not in-memory fakes)
[ ] Are test fixtures using builders? (not raw constructors)
[ ] Is test data isolated? (@BeforeEach cleanup)
[ ] Are parameterized tests used for multiple input variants?
[ ] Are flaky tests absent? (no Thread.sleep, use Awaitility)
      ❌ Thread.sleep(5000); assertThat(result).isNotNull();
      ✅ await().atMost(10, SECONDS).untilAsserted(() -> assertThat(...));
[ ] Is the new code's mutation testing score adequate?
```

---

## 📊 Section 8: Observability

```
[ ] Are business events logged at INFO level?
[ ] Are errors logged with full context and stack trace?
[ ] Is correlationId present in log MDC?
[ ] Are metrics emitted for key operations?
      • Counter for outcomes (success/failure)
      • Timer for external call durations
      • Gauge for queue depths, pool sizes
[ ] Are metric tags LOW cardinality? (never userId, paymentId)
[ ] Is PII masked in all log output?
[ ] Are structured log fields consistent with team conventions?
[ ] Are new alert rules needed for this change?
```

---

## 📄 Section 9: PR Quality

```
[ ] Is the PR description clear? (what, why, how)
[ ] Is the PR a reasonable size? (< 400 lines changed ideally)
[ ] Are large PRs broken into reviewable commits?
[ ] Are configuration changes documented?
[ ] Is the migration plan documented? (schema changes, feature flags)
[ ] Are breaking changes called out explicitly?
[ ] Is rollback plan considered?
[ ] Are environment-specific changes flagged? (prod config, feature toggles)
```

---

## 🚫 Instant Rejection Triggers

These should NEVER pass code review:

```
🔴 Secrets hardcoded in source code
🔴 No error handling on external calls (timeout, retry, circuit breaker)
🔴 Swallowed exceptions (empty catch blocks)
🔴 No tests for new business logic
🔴 Dual-write without outbox pattern (DB + Kafka without atomicity)
🔴 Auto-commit enabled for Kafka consumers processing critical data
🔴 PII logged in plain text
🔴 Unbounded collections loaded into memory
🔴 synchronized blocks in code using virtual threads
🔴 Thread.sleep() in tests instead of Awaitility
🔴 Field injection (@Autowired on fields)
🔴 Domain layer importing Spring/framework classes
```

---

## ⚡ Reviewer Response Templates

### Requesting Changes

```
"This looks good overall! A few things to address before merging:"

"Blocking: [Correctness/Security issue] — The retry logic applies to
non-idempotent operations. This could cause duplicate payments.
Suggest: Add idempotency key check or restrict retry to GET operations."

"Non-blocking: [Style] — Consider extracting this validation into a
separate method for readability. Happy to merge as-is if you prefer."
```

### Approval with Minor Notes

```
"LGTM! ✅ Clean implementation. One optional suggestion for a follow-up:
consider adding a metric for gateway call duration (non-blocking)."
```

---

## 💡 Golden Rules of Code Review

```
1.  REVIEW THE CHANGE, not the person — be kind, be specific, be constructive.
2.  CORRECTNESS > STYLE — catch bugs first, bikeshed formatting never.
3.  ASK QUESTIONS before assuming — "What happens if X?" > "This is wrong."
4.  PRAISE GOOD CODE — reinforcement works better than criticism alone.
5.  SMALL PRs get better reviews — encourage smaller, focused changes.
6.  AUTOMATE what you can — formatting (checkstyle), architecture (ArchUnit), security (SAST).
7.  The GOAL is shared understanding — both reviewer and author should learn.
8.  BLOCKING = "This will cause a bug/security issue/outage if deployed."
9.  NON-BLOCKING = "This could be improved but is safe to merge as-is."
10. If you're unsure, PAIR on it — 5 minutes of conversation > 10 comment threads.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / MongoDB / Kafka / Resilience4j*
