# ☕ Modern Java Features Quick Reference (Java 17-24+)

> **Purpose:** Concise lookup card for all modern Java features with practical examples. Reference when using sealed types, pattern matching, records, virtual threads, and other post-Java-16 features.

---

## 📦 Records (Java 16+)

```java
// Immutable data carrier — replaces boilerplate POJOs
public record Money(BigDecimal value, String currency) {
    // Compact constructor — validation
    public Money {
        Objects.requireNonNull(value);
        Objects.requireNonNull(currency);
        value = value.setScale(2, RoundingMode.HALF_UP);
    }
    // Static factory
    public static Money of(String amount, String currency) {
        return new Money(new BigDecimal(amount), currency);
    }
    // Custom methods
    public Money add(Money other) {
        return new Money(value.add(other.value), currency);
    }
}

// Local records (method-scoped, Java 16+)
List<RankedPayment> ranked = payments.stream()
    .map(p -> { record Scored(Payment payment, int score) {} return new Scored(p, score(p)); })
    .sorted(Comparator.comparingInt(Scored::score).reversed())
    .toList();
```

---

## 🔒 Sealed Types (Java 17+)

```java
// Controlled hierarchy — compiler enforces exhaustiveness
public sealed interface PaymentEvent permits
    PaymentEvent.Initiated, PaymentEvent.Captured, PaymentEvent.Failed, PaymentEvent.Refunded {

    String paymentId();

    record Initiated(String paymentId, Money amount, String customerId) implements PaymentEvent {}
    record Captured(String paymentId, Money settled, String txId) implements PaymentEvent {}
    record Failed(String paymentId, String reason, boolean retryable) implements PaymentEvent {}
    record Refunded(String paymentId, Money amount, String reason) implements PaymentEvent {}
}

// Exhaustive switch — compiler error if case missing
String describe(PaymentEvent event) {
    return switch (event) {
        case PaymentEvent.Initiated e -> "Payment %s initiated for %s".formatted(e.paymentId(), e.amount());
        case PaymentEvent.Captured e  -> "Payment %s captured: %s".formatted(e.paymentId(), e.txId());
        case PaymentEvent.Failed e    -> "Payment %s failed: %s".formatted(e.paymentId(), e.reason());
        case PaymentEvent.Refunded e  -> "Payment %s refunded: %s".formatted(e.paymentId(), e.amount());
    }; // No default needed — compiler ensures exhaustiveness
}
```

---

## 🎯 Pattern Matching (Java 21+)

```java
// instanceof pattern
if (event instanceof PaymentEvent.Failed f && f.retryable()) {
    retryQueue.enqueue(f.paymentId());
}

// Switch with patterns and guards
String handleEvent(PaymentEvent event) {
    return switch (event) {
        case PaymentEvent.Failed f when f.retryable()  -> retry(f);
        case PaymentEvent.Failed f                      -> alertAndDLT(f);
        case PaymentEvent.Captured c when isHighValue(c) -> flagForReview(c);
        case PaymentEvent.Captured c                    -> acknowledge(c);
        default                                         -> log(event);
    };
}

// Record deconstruction (Java 21+)
switch (event) {
    case PaymentEvent.Initiated(var id, var amount, var customer)
        when amount.value().compareTo(BigDecimal.valueOf(10000)) > 0 ->
            flagHighValue(id, amount);
    default -> process(event);
}

// Unnamed patterns (Java 22+)
switch (event) {
    case PaymentEvent.Initiated _ -> handleNew();     // Don't need the binding
    case PaymentEvent.Failed _    -> handleFailure();
    default -> {}
}
```

---

## 🧵 Virtual Threads (Java 21+)

```java
// One-liner in Spring Boot
// spring.threads.virtual.enabled: true

// Manual creation
Thread.startVirtualThread(() -> blockingIoCall());

// Executor (preferred)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = tasks.stream().map(t -> executor.submit(t)).toList();
    futures.forEach(f -> f.get(10, TimeUnit.SECONDS));
}

// ⚠️ Avoid: synchronized (pins carrier), ThreadLocal (memory per VT)
// ✅ Use: ReentrantLock, ScopedValue
```

---

## 📝 Text Blocks (Java 15+)

```java
String json = """
    {
        "paymentId": "%s",
        "amount": %s,
        "status": "CAPTURED"
    }
    """.formatted(paymentId, amount);

String sql = """
    SELECT p.id, p.amount, c.name
    FROM payments p
    JOIN customers c ON p.customer_id = c.id
    WHERE p.status = 'CAPTURED'
    ORDER BY p.created_at DESC
    """;
```

---

## 🔀 Switch Expressions (Java 14+)

```java
// Expression — returns a value
BigDecimal feeRate = switch (paymentType) {
    case CREDIT_CARD -> new BigDecimal("0.029");
    case ACH         -> new BigDecimal("0.001");
    case WIRE        -> new BigDecimal("0.0005");
};

// Yield for blocks
String message = switch (status) {
    case CAPTURED -> "Payment successful";
    case FAILED -> {
        log.warn("Payment failed");
        yield "Payment could not be processed";
    }
    default -> "Payment status: " + status;
};
```

---

## 📚 Sequenced Collections (Java 21+)

```java
SequencedCollection<Payment> payments = new ArrayList<>();
payments.addFirst(urgentPayment);     // Add to beginning
payments.addLast(normalPayment);      // Add to end
Payment first = payments.getFirst();  // Get first element
Payment last = payments.getLast();    // Get last element
payments.reversed().forEach(...);     // Iterate in reverse

SequencedMap<String, Payment> map = new LinkedHashMap<>();
map.firstEntry();    // First key-value pair
map.lastEntry();     // Last key-value pair
map.pollLastEntry(); // Remove and return last
```

---

## 🏷️ Unnamed Variables (Java 22+)

```java
// Unused lambda parameters
payments.forEach(_ -> processedCount.increment());

// Unused catch variable
try { parse(json); } catch (JsonException _) { return defaultValue; }

// Unused pattern binding
if (obj instanceof Payment _) { log.info("Is a payment"); }
```

---

## 🔧 Helpful NullPointerExceptions (Java 17+)

```java
// Before: "NullPointerException" (where??)
// After:  "Cannot invoke Payment.amount() because payment is null"
//         "Cannot invoke Money.value() because the return value of Payment.amount() is null"
// Enabled by default in Java 17+
```

---

## 📊 Stream Additions

```java
// toList() (Java 16+) — unmodifiable list
var ids = payments.stream().map(Payment::id).toList();

// mapMulti (Java 16+) — conditional flat mapping
payments.stream().<String>mapMulti((p, consumer) -> {
    if (p.status() == CAPTURED) consumer.accept(p.id());
}).toList();

// Gatherers (Java 24+ preview) — custom intermediate ops
payments.stream()
    .gather(Gatherers.windowFixed(100))  // Batch into groups of 100
    .forEach(batch -> processBatch(batch));
```

---

## 🏭 Collections Factories

```java
// Unmodifiable collections (Java 9+)
var list = List.of("a", "b", "c");
var set = Set.of(1, 2, 3);
var map = Map.of("key1", "val1", "key2", "val2");
var map2 = Map.ofEntries(entry("k1", "v1"), entry("k2", "v2"));

// Copies (Java 10+)
var copy = List.copyOf(mutableList);   // Unmodifiable copy
var setCopy = Set.copyOf(mutableSet);
```

---

## Quick Lookup: Feature → Java Version

| Feature | Version | Status |
|---------|---------|--------|
| Records | **16** | Stable |
| Sealed classes | **17** | Stable |
| Pattern matching instanceof | **16** | Stable |
| Switch pattern matching | **21** | Stable |
| Record patterns (deconstruction) | **21** | Stable |
| Virtual threads | **21** | Stable |
| Sequenced collections | **21** | Stable |
| Unnamed variables `_` | **22** | Stable |
| Structured concurrency | **24** | Preview |
| Scoped values | **24** | Preview |
| Stream gatherers | **24** | Preview |
| String templates | Withdrawn | — |

---

*Last updated: February 2026 | Java 21+ focus*
