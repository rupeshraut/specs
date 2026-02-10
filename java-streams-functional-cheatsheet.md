# λ Java Streams & Functional Programming Cheat Sheet

> **Purpose:** Production patterns for Stream API, collectors, Optional, and functional composition. Reference before writing any data transformation pipeline.
> **Stack context:** Java 21+ / Records / Pattern Matching

---

## 📋 Stream Decision Framework

| Scenario | Use |
|----------|-----|
| Transform collection elements | `.map()` |
| Filter by condition | `.filter()` |
| Flatten nested collections | `.flatMap()` |
| Side effects (logging, metrics) | `.peek()` (debug only) or `forEach` (terminal) |
| Reduce to single value | `.reduce()` or `.collect()` |
| Group by property | `Collectors.groupingBy()` |
| Check existence | `.anyMatch()` / `.allMatch()` / `.noneMatch()` |
| Find one element | `.findFirst()` / `.findAny()` |
| Need index or state? | DON'T use streams — use for loop |

---

## 🔄 Pattern 1: Core Operations

### Map, Filter, FlatMap

```java
// ── Transform ──
List<String> paymentIds = payments.stream()
    .map(Payment::id)
    .toList();

// ── Filter + Transform ──
List<Money> capturedAmounts = payments.stream()
    .filter(p -> p.status() == PaymentStatus.CAPTURED)
    .map(Payment::amount)
    .toList();

// ── FlatMap: flatten nested collections ──
// Each customer has List<Payment>, get all payments across all customers
List<Payment> allPayments = customers.stream()
    .flatMap(c -> c.payments().stream())
    .toList();

// ── FlatMap with Optional: skip empties ──
List<String> transactionIds = payments.stream()
    .map(Payment::gatewayResponse)     // Optional<GatewayResponse>
    .flatMap(Optional::stream)          // Skip empty Optionals
    .map(GatewayResponse::transactionId)
    .toList();

// ── mapMulti (Java 16+): conditional expansion ──
List<AuditEntry> entries = payments.stream()
    .<AuditEntry>mapMulti((payment, consumer) -> {
        consumer.accept(new AuditEntry(payment.id(), "CREATED", payment.createdAt()));
        if (payment.status() == PaymentStatus.CAPTURED) {
            consumer.accept(new AuditEntry(payment.id(), "CAPTURED", payment.updatedAt()));
        }
    })
    .toList();
```

### Reduce

```java
// ── Sum amounts ──
BigDecimal total = payments.stream()
    .map(p -> p.amount().value())
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// ── Find max ──
Optional<Payment> largest = payments.stream()
    .max(Comparator.comparing(p -> p.amount().value()));

// ── Combine strings ──
String summary = payments.stream()
    .map(p -> "%s: %s".formatted(p.id(), p.status()))
    .collect(Collectors.joining(", ", "[", "]"));
// → "[pay-1: CAPTURED, pay-2: FAILED, pay-3: PENDING]"
```

---

## 📊 Pattern 2: Collectors

### Grouping

```java
// ── Group by status ──
Map<PaymentStatus, List<Payment>> byStatus = payments.stream()
    .collect(Collectors.groupingBy(Payment::status));

// ── Group by status, count each ──
Map<PaymentStatus, Long> countByStatus = payments.stream()
    .collect(Collectors.groupingBy(Payment::status, Collectors.counting()));

// ── Group by status, sum amounts ──
Map<PaymentStatus, BigDecimal> totalByStatus = payments.stream()
    .collect(Collectors.groupingBy(
        Payment::status,
        Collectors.reducing(BigDecimal.ZERO,
            p -> p.amount().value(), BigDecimal::add)));

// ── Nested grouping: by status then by type ──
Map<PaymentStatus, Map<PaymentType, List<Payment>>> nested = payments.stream()
    .collect(Collectors.groupingBy(Payment::status,
        Collectors.groupingBy(Payment::type)));

// ── Group by, then transform values ──
Map<String, List<String>> paymentIdsByCustomer = payments.stream()
    .collect(Collectors.groupingBy(
        Payment::customerId,
        Collectors.mapping(Payment::id, Collectors.toList())));
```

### Partitioning & Collecting to Maps

```java
// ── Partition: exactly two groups (true/false) ──
Map<Boolean, List<Payment>> partitioned = payments.stream()
    .collect(Collectors.partitioningBy(p -> p.amount().value().compareTo(new BigDecimal("1000")) > 0));
List<Payment> highValue = partitioned.get(true);
List<Payment> normalValue = partitioned.get(false);

// ── toMap ──
Map<String, Payment> paymentById = payments.stream()
    .collect(Collectors.toMap(Payment::id, Function.identity()));

// ── toMap with merge (handle duplicate keys) ──
Map<String, Payment> latestByCustomer = payments.stream()
    .collect(Collectors.toMap(
        Payment::customerId,
        Function.identity(),
        (existing, replacement) ->
            existing.createdAt().isAfter(replacement.createdAt()) ? existing : replacement));

// ── toUnmodifiableMap ──
Map<String, PaymentStatus> statusMap = payments.stream()
    .collect(Collectors.toUnmodifiableMap(Payment::id, Payment::status));

// ── Collecting to EnumMap (faster for enum keys) ──
EnumMap<PaymentStatus, List<Payment>> enumGrouped = payments.stream()
    .collect(Collectors.groupingBy(Payment::status,
        () -> new EnumMap<>(PaymentStatus.class), Collectors.toList()));
```

### Statistics & Summarizing

```java
// ── Summary statistics ──
DoubleSummaryStatistics stats = payments.stream()
    .mapToDouble(p -> p.amount().value().doubleValue())
    .summaryStatistics();
// stats.getCount(), getSum(), getMin(), getMax(), getAverage()

// ── Custom summarization ──
record PaymentSummary(long count, BigDecimal total, BigDecimal average, BigDecimal max) {}

PaymentSummary summary = payments.stream()
    .collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.reducing(BigDecimal.ZERO, p -> p.amount().value(), BigDecimal::add),
        (count, total) -> new PaymentSummary(
            count, total,
            count > 0 ? total.divide(BigDecimal.valueOf(count), 2, RoundingMode.HALF_UP) : BigDecimal.ZERO,
            payments.stream().map(p -> p.amount().value()).max(Comparator.naturalOrder()).orElse(BigDecimal.ZERO)
        )
    ));
```

### Custom Collector

```java
// ── Batch collector: split stream into fixed-size batches ──
public static <T> Collector<T, ?, List<List<T>>> batching(int batchSize) {
    return Collector.of(
        ArrayList::new,
        (batches, item) -> {
            if (batches.isEmpty() || batches.getLast().size() >= batchSize) {
                batches.add(new ArrayList<>());
            }
            batches.getLast().add(item);
        },
        (left, right) -> { left.addAll(right); return left; }
    );
}

// Usage
List<List<Payment>> batches = payments.stream()
    .collect(batching(100));
// → [[pay1..pay100], [pay101..pay200], ...]
```

---

## 🔗 Pattern 3: Optional

### Good Optional Patterns

```java
// ── Return type for "might not exist" ──
public Optional<Payment> findById(String id) { ... }

// ── Chain transformations ──
String customerName = paymentRepository.findById(id)
    .map(Payment::customerId)
    .flatMap(customerRepository::findById)   // Returns Optional<Customer>
    .map(Customer::name)
    .orElse("Unknown");

// ── Conditional action ──
paymentRepository.findById(id)
    .filter(p -> p.status() == PaymentStatus.PENDING)
    .ifPresent(p -> processor.process(p));

// ── orElseThrow (for mandatory lookup) ──
Payment payment = repository.findById(id)
    .orElseThrow(() -> new PaymentNotFoundException(id));

// ── or(): fallback to another Optional (Java 9+) ──
Payment payment = primaryRepo.findById(id)
    .or(() -> secondaryRepo.findById(id))
    .or(() -> archiveRepo.findById(id))
    .orElseThrow(() -> new PaymentNotFoundException(id));

// ── stream(): integrate Optional into Stream pipeline ──
List<Payment> found = paymentIds.stream()
    .map(repository::findById)      // Stream<Optional<Payment>>
    .flatMap(Optional::stream)       // Stream<Payment> — skips empties
    .toList();

// ── ifPresentOrElse (Java 9+) ──
repository.findById(id).ifPresentOrElse(
    payment -> log.info("Found payment: {}", payment.id()),
    () -> log.warn("Payment not found: {}", id)
);
```

### Optional Anti-Patterns

```java
// ❌ Optional as field
public class Payment {
    private Optional<String> description;  // Use @Nullable String instead
}

// ❌ Optional as method parameter
public void process(Optional<FraudResult> result) { }  // Use @Nullable or overload

// ❌ Optional.get() without check
var payment = repository.findById(id).get();  // NoSuchElementException!

// ❌ isPresent + get (verbose)
if (result.isPresent()) { return result.get().amount(); }

// ✅ Use map/flatMap/orElse instead
return result.map(PaymentResult::amount).orElse(Money.ZERO);

// ❌ Optional.of(null) — throws NPE
Optional.of(nullableValue);  // 💥

// ✅ Use ofNullable for potentially null values
Optional.ofNullable(nullableValue);
```

---

## 🧱 Pattern 4: Functional Composition

### Function Composition

```java
// ── Compose transformations ──
Function<Payment, Money> extractAmount = Payment::amount;
Function<Money, BigDecimal> extractValue = Money::value;
Function<BigDecimal, String> format = v -> "$" + v.setScale(2, RoundingMode.HALF_UP);

Function<Payment, String> formatPaymentAmount =
    extractAmount.andThen(extractValue).andThen(format);

String display = formatPaymentAmount.apply(payment);  // "$100.00"

// ── Predicate composition ──
Predicate<Payment> isHighValue = p -> p.amount().value().compareTo(new BigDecimal("10000")) > 0;
Predicate<Payment> isCreditCard = p -> p.type() == PaymentType.CREDIT_CARD;
Predicate<Payment> isPending = p -> p.status() == PaymentStatus.PENDING;

Predicate<Payment> needsReview = isHighValue.and(isCreditCard).and(isPending);
Predicate<Payment> skipReview = needsReview.negate();

List<Payment> forReview = payments.stream().filter(needsReview).toList();
```

### Functional Error Handling (Result Type)

```java
// ── Result<T> instead of throwing exceptions ──
public sealed interface Result<T> {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error, String code) implements Result<T> {}

    default <R> Result<R> map(Function<T, R> fn) {
        return switch (this) {
            case Success<T> s -> new Success<>(fn.apply(s.value()));
            case Failure<T> f -> new Failure<>(f.error(), f.code());
        };
    }

    default <R> Result<R> flatMap(Function<T, Result<R>> fn) {
        return switch (this) {
            case Success<T> s -> fn.apply(s.value());
            case Failure<T> f -> new Failure<>(f.error(), f.code());
        };
    }

    default T orElseThrow() {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> throw new BusinessException(f.code(), f.error());
        };
    }
}

// Usage: chain operations that might fail
Result<PaymentReceipt> result = validatePayment(request)
    .flatMap(this::checkFraud)
    .flatMap(this::chargeGateway)
    .map(this::toReceipt);
```

---

## 🚫 Stream Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Stream for side effects only** | Unclear intent, hard to debug | Use for-each loop |
| **Nested streams** (stream in stream) | O(n²), hard to read | Flatten with flatMap or collect to Map first |
| **parallelStream() for I/O** | Blocks ForkJoinPool threads | Virtual thread executor |
| **Modifying source during stream** | ConcurrentModificationException | Collect to new collection |
| **Optional as field/parameter** | Misuse of Optional's purpose | @Nullable or overloaded methods |
| **Stream reuse** | IllegalStateException on second use | Create new stream each time |
| **Complex reduce** | Unreadable, error-prone | Custom collector or for loop |
| **.peek() in production** | Not guaranteed to execute, debugging only | Remove before committing |
| **Ignoring stream order** | Wrong results with findFirst/limit | Use sorted() explicitly |

---

## 💡 Golden Rules of Functional Java

```
1.  STREAMS for transformation — for-each for side effects.
2.  Optional is a RETURN TYPE — never a field, parameter, or collection element.
3.  IMMUTABLE data in, immutable data out — streams don't mutate, they transform.
4.  flatMap FLATTENS — Optional<Optional<T>> → Optional<T>, Stream<Stream<T>> → Stream<T>.
5.  Collectors.groupingBy is your Swiss army knife — combine with counting, mapping, reducing.
6.  COMPOSE predicates and functions — build complex behavior from simple pieces.
7.  parallelStream is RARELY the answer — use virtual threads for I/O parallelism.
8.  If it needs an index or break/continue — use a for loop, not a stream.
9.  Keep stream pipelines SHORT — extract complex operations into named methods.
10. toList() (Java 16+) > collect(Collectors.toList()) — shorter, returns unmodifiable.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Records / Streams / Optional*
