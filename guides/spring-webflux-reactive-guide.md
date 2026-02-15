# Spring WebFlux & Reactive Programming

A comprehensive guide to reactive programming with Spring WebFlux, Project Reactor, and reactive patterns for high-performance non-blocking applications.

---

## Table of Contents

1. [Reactive Fundamentals](#reactive-fundamentals)
2. [When to Use Reactive](#when-to-use-reactive)
3. [Project Reactor Basics](#project-reactor-basics)
4. [Mono vs Flux](#mono-vs-flux)
5. [WebFlux Controllers](#webflux-controllers)
6. [Reactive WebClient](#reactive-webclient)
7. [Reactive Data Access](#reactive-data-access)
8. [Error Handling](#error-handling)
9. [Backpressure](#backpressure)
10. [Testing Reactive Code](#testing-reactive-code)
11. [Performance Tuning](#performance-tuning)
12. [Common Pitfalls](#common-pitfalls)

---

## Reactive Fundamentals

### Reactive Manifesto

```
Responsive: Quick response times
Resilient: Stay responsive under failure
Elastic: Scale under varying load
Message-Driven: Async message passing
```

### Blocking vs Non-Blocking

```
Blocking (Traditional):
Thread ──request──> [Waiting for DB] ──response──> Client
         (blocked)                     (wasted)

Non-Blocking (Reactive):
Thread ──request──> [DB query starts] ──continue other work──
       ──callback──> [DB done] ──response──> Client
         (freed)               (efficient)

Benefits:
✅ Higher throughput with fewer threads
✅ Better resource utilization
✅ Handles backpressure
✅ Natural composition of async operations

Costs:
⚠️  Steeper learning curve
⚠️  Debugging complexity
⚠️  Requires reactive all the way down
⚠️  Context switching overhead
```

---

## When to Use Reactive

### ✅ Good Use Cases

```java
// 1. High concurrency I/O-bound workloads
// - API gateway
// - Microservices with many external calls
// - Event streaming

// 2. Real-time data streaming
// - Live dashboards
// - Chat applications
// - Stock tickers

// 3. Integration with reactive systems
// - Kafka
// - MongoDB Reactive
// - Redis Reactive
```

### ❌ When NOT to Use Reactive

```java
// 1. CPU-intensive operations
// - Heavy calculations
// - Image processing
// - Data transformation

// 2. Simple CRUD applications
// - Low traffic
// - Simple business logic
// - Blocking dependencies (JDBC)

// 3. Team lacks reactive experience
// - Maintenance concerns
// - Debugging difficulties
// - Higher complexity
```

### Decision Matrix

| Factor | Use WebMVC | Use WebFlux |
|--------|-----------|-------------|
| **Traffic** | Low-Medium | High |
| **I/O Operations** | Few | Many parallel calls |
| **Blocking Libraries** | Yes (JDBC) | No (R2DBC) |
| **Team Experience** | Reactive = New | Reactive = Experienced |
| **Real-time Needs** | No | Yes (SSE, WebSocket) |
| **Latency Requirements** | Normal | Low latency critical |

---

## Project Reactor Basics

### Mono - 0 or 1 element

```java
// Creating Mono
Mono<String> emptyMono = Mono.empty();
Mono<String> justMono = Mono.just("Hello");
Mono<String> errorMono = Mono.error(new RuntimeException("Error"));
Mono<String> deferredMono = Mono.defer(() -> Mono.just(getValue()));
Mono<String> fromCallable = Mono.fromCallable(() -> expensiveOperation());

// Transforming
Mono<String> transformed = Mono.just("hello")
    .map(String::toUpperCase)              // HELLO
    .filter(s -> s.length() > 3)           // HELLO
    .defaultIfEmpty("default");            // HELLO

// Subscribing
mono.subscribe(
    value -> System.out.println("Got: " + value),
    error -> System.err.println("Error: " + error),
    () -> System.out.println("Completed")
);
```

### Flux - 0 to N elements

```java
// Creating Flux
Flux<Integer> range = Flux.range(1, 10);
Flux<String> fromArray = Flux.fromArray(new String[]{"a", "b", "c"});
Flux<String> fromIterable = Flux.fromIterable(List.of("x", "y", "z"));
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));

// Transforming
Flux<String> transformed = Flux.just("apple", "banana", "cherry")
    .map(String::toUpperCase)              // APPLE, BANANA, CHERRY
    .filter(s -> s.startsWith("B"))        // BANANA
    .take(5)                               // First 5
    .skip(2);                              // Skip first 2

// Combining
Flux<String> merged = Flux.merge(flux1, flux2);
Flux<String> concatenated = Flux.concat(flux1, flux2);
Flux<Tuple2<String, Integer>> zipped = Flux.zip(names, ages);
```

---

## WebFlux Controllers

### Basic Controller

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final PaymentService paymentService;

    // Return Mono for single object
    @GetMapping("/{id}")
    public Mono<Payment> getPayment(@PathVariable String id) {
        return paymentService.findById(id);
    }

    // Return Flux for collection
    @GetMapping
    public Flux<Payment> getAllPayments() {
        return paymentService.findAll();
    }

    // Create payment
    @PostMapping
    public Mono<Payment> createPayment(@RequestBody PaymentRequest request) {
        return paymentService.create(request);
    }

    // Streaming response
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Payment> streamPayments() {
        return paymentService.streamPayments()
            .delayElements(Duration.ofSeconds(1));
    }
}
```

### Error Handling

```java
@RestController
public class ReactivePaymentController {

    @GetMapping("/payments/{id}")
    public Mono<Payment> getPayment(@PathVariable String id) {
        return paymentService.findById(id)
            // Handle specific error
            .onErrorResume(PaymentNotFoundException.class, e ->
                Mono.error(new ResponseStatusException(
                    HttpStatus.NOT_FOUND, e.getMessage()
                ))
            )
            // Handle any error
            .onErrorResume(e ->
                Mono.error(new ResponseStatusException(
                    HttpStatus.INTERNAL_SERVER_ERROR,
                    "Error processing payment"
                ))
            )
            // Fallback value
            .onErrorReturn(Payment.empty())
            // Retry on failure
            .retry(3)
            // Retry with backoff
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }
}
```

---

## Reactive WebClient

### Configuration

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentWebClient() {
        return WebClient.builder()
            .baseUrl("https://payment-api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.USER_AGENT, "payment-service/1.0")
            .filter(logRequest())
            .filter(logResponse())
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                    .responseTimeout(Duration.ofSeconds(30))
                    .wiretap(true)
            ))
            .build();
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.info("Request: {} {}", request.method(), request.url());
            return Mono.just(request);
        });
    }
}
```

### Making Requests

```java
@Service
public class PaymentApiClient {

    private final WebClient webClient;

    // GET request
    public Mono<Payment> getPayment(String id) {
        return webClient.get()
            .uri("/payments/{id}", id)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response ->
                Mono.error(new PaymentNotFoundException(id))
            )
            .onStatus(HttpStatus::is5xxServerError, response ->
                Mono.error(new PaymentServiceException())
            )
            .bodyToMono(Payment.class)
            .timeout(Duration.ofSeconds(5))
            .retry(3);
    }

    // POST request
    public Mono<Payment> createPayment(PaymentRequest request) {
        return webClient.post()
            .uri("/payments")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(Payment.class);
    }

    // Streaming response
    public Flux<PaymentEvent> streamPaymentEvents() {
        return webClient.get()
            .uri("/payments/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(PaymentEvent.class);
    }

    // Parallel requests
    public Mono<PaymentSummary> getPaymentSummary(String userId) {
        Mono<User> userMono = getUserById(userId);
        Mono<List<Payment>> paymentsMono = getPaymentsByUser(userId).collectList();
        Mono<Balance> balanceMono = getUserBalance(userId);

        return Mono.zip(userMono, paymentsMono, balanceMono)
            .map(tuple -> new PaymentSummary(
                tuple.getT1(), // user
                tuple.getT2(), // payments
                tuple.getT3()  // balance
            ));
    }
}
```

---

## Reactive Data Access

### R2DBC with PostgreSQL

```java
@Configuration
@EnableR2dbcRepositories
public class R2dbcConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        return new PostgresqlConnectionFactory(
            PostgresqlConnectionConfiguration.builder()
                .host("localhost")
                .port(5432)
                .database("payments")
                .username("user")
                .password("password")
                .build()
        );
    }
}
```

### Reactive Repository

```java
public interface PaymentRepository extends R2dbcRepository<Payment, String> {

    Flux<Payment> findByUserId(String userId);

    Flux<Payment> findByStatus(PaymentStatus status);

    @Query("SELECT * FROM payments WHERE amount > :amount")
    Flux<Payment> findLargePayments(BigDecimal amount);
}

@Service
public class PaymentService {

    private final PaymentRepository repository;

    public Mono<Payment> create(PaymentRequest request) {
        Payment payment = Payment.from(request);
        return repository.save(payment);
    }

    public Flux<Payment> findByUser(String userId) {
        return repository.findByUserId(userId)
            .filter(payment -> payment.getStatus() != PaymentStatus.CANCELLED);
    }

    public Mono<Void> deleteOldPayments() {
        return repository.findByStatus(PaymentStatus.COMPLETED)
            .filter(payment -> payment.isOlderThan(Duration.ofDays(90)))
            .flatMap(repository::delete)
            .then();
    }
}
```

### Reactive MongoDB

```java
public interface PaymentRepository extends ReactiveMongoRepository<Payment, String> {

    Flux<Payment> findByUserId(String userId);

    Flux<Payment> findByCreatedAtAfter(Instant timestamp);
}

@Service
public class ReactivePaymentService {

    private final PaymentRepository repository;

    // Sequential chaining
    public Mono<Payment> processPayment(PaymentRequest request) {
        return validatePayment(request)
            .flatMap(this::checkBalance)
            .flatMap(this::deductAmount)
            .flatMap(repository::save)
            .flatMap(this::sendNotification);
    }

    // Parallel execution
    public Mono<PaymentReport> generateReport(String userId) {
        Mono<Long> totalCount = repository.countByUserId(userId);
        Mono<BigDecimal> totalAmount = repository.findByUserId(userId)
            .map(Payment::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        Mono<Payment> latestPayment = repository.findByUserId(userId)
            .sort(Comparator.comparing(Payment::getCreatedAt).reversed())
            .next();

        return Mono.zip(totalCount, totalAmount, latestPayment)
            .map(tuple -> new PaymentReport(
                tuple.getT1(), // count
                tuple.getT2(), // total
                tuple.getT3()  // latest
            ));
    }
}
```

---

## Backpressure

### Understanding Backpressure

```java
// Producer is faster than consumer
Flux.range(1, 1000000)  // Fast producer
    .delayElements(Duration.ofMillis(1))  // Slow consumer
    .subscribe(System.out::println);

// Without backpressure → OutOfMemoryError
// With backpressure → Producer slows down
```

### Backpressure Strategies

```java
@Service
public class BackpressureExamples {

    // 1. Buffer - queue elements
    public Flux<Event> bufferStrategy() {
        return eventSource()
            .buffer(100)  // Buffer up to 100 elements
            .flatMap(Flux::fromIterable);
    }

    // 2. Drop - discard excess elements
    public Flux<Event> dropStrategy() {
        return eventSource()
            .onBackpressureDrop(event ->
                log.warn("Dropped event: {}", event)
            );
    }

    // 3. Latest - keep only latest
    public Flux<Event> latestStrategy() {
        return eventSource()
            .onBackpressureLatest();
    }

    // 4. Error - fail fast
    public Flux<Event> errorStrategy() {
        return eventSource()
            .onBackpressureError();
    }

    // 5. Window - process in batches
    public Flux<List<Event>> windowStrategy() {
        return eventSource()
            .window(Duration.ofSeconds(1))
            .flatMap(window -> window.collectList());
    }
}
```

---

## Testing Reactive Code

### StepVerifier

```java
@Test
void testPaymentCreation() {
    PaymentRequest request = new PaymentRequest(/*...*/);

    Mono<Payment> result = paymentService.create(request);

    StepVerifier.create(result)
        .assertNext(payment -> {
            assertThat(payment.getId()).isNotNull();
            assertThat(payment.getStatus()).isEqualTo(PaymentStatus.PENDING);
            assertThat(payment.getAmount()).isEqualTo(request.getAmount());
        })
        .verifyComplete();
}

@Test
void testPaymentStream() {
    Flux<Payment> stream = paymentService.streamPayments();

    StepVerifier.create(stream.take(3))
        .expectNextCount(3)
        .verifyComplete();
}

@Test
void testErrorHandling() {
    Mono<Payment> result = paymentService.findById("invalid-id");

    StepVerifier.create(result)
        .expectError(PaymentNotFoundException.class)
        .verify();
}

@Test
void testWithVirtualTime() {
    StepVerifier.withVirtualTime(() ->
        Flux.interval(Duration.ofHours(1)).take(3)
    )
        .expectSubscription()
        .expectNoEvent(Duration.ofHours(1))
        .expectNext(0L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(1L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(2L)
        .verifyComplete();
}
```

---

## Common Pitfalls

### 1. Blocking in Reactive Chain

```java
// ❌ BAD: Blocking operation
Mono<Payment> payment = paymentService.findById(id)
    .map(p -> {
        // Blocking!
        String result = blockingHttpCall();
        return p.withResult(result);
    });

// ✅ GOOD: Non-blocking alternative
Mono<Payment> payment = paymentService.findById(id)
    .flatMap(p ->
        webClient.get()
            .uri("/external")
            .retrieve()
            .bodyToMono(String.class)
            .map(result -> p.withResult(result))
    );
```

### 2. Not Subscribing

```java
// ❌ BAD: Nothing happens (cold publisher)
Mono<Payment> payment = paymentService.create(request);
// No subscription = no execution!

// ✅ GOOD: Subscribe or return
paymentService.create(request)
    .subscribe(); // Triggers execution

// Or let framework subscribe
@GetMapping
public Mono<Payment> create(@RequestBody PaymentRequest request) {
    return paymentService.create(request); // Framework subscribes
}
```

### 3. Nested Subscribes

```java
// ❌ BAD: Nested subscribes
paymentService.findById(id).subscribe(payment -> {
    notificationService.send(payment).subscribe(result -> {
        // Reactive hell!
    });
});

// ✅ GOOD: Flat composition
paymentService.findById(id)
    .flatMap(notificationService::send)
    .subscribe(result -> {
        // Clean!
    });
```

---

## Best Practices

### ✅ DO

- Use `flatMap` for async transformations
- Use `map` for synchronous transformations
- Handle errors explicitly with `onErrorResume`
- Use `StepVerifier` for testing
- Leverage backpressure strategies
- Use WebClient instead of RestTemplate
- Return Mono/Flux from controllers
- Use reactive database drivers (R2DBC, Reactive MongoDB)

### ❌ DON'T

- Block inside reactive chains
- Use `.block()` or `.blockFirst()` in production
- Nest subscribes
- Mix blocking and reactive code
- Ignore backpressure
- Use JDBC with WebFlux
- Forget to subscribe
- Swallow errors silently

---

*This guide provides comprehensive patterns for building reactive applications with Spring WebFlux. Reactive programming offers significant benefits for I/O-intensive applications but requires a different mindset and careful implementation.*
