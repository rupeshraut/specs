# Effective Modern Project Reactor

A comprehensive guide to mastering reactive programming with Project Reactor, building non-blocking applications with Mono and Flux.

---

## Table of Contents

1. [Reactor Fundamentals](#reactor-fundamentals)
2. [Mono\<T\>](#monot)
3. [Flux\<T\>](#fluxt)
4. [Creating Publishers](#creating-publishers)
5. [Transformation Operators](#transformation-operators)
6. [Filtering Operators](#filtering-operators)
7. [Combining Publishers](#combining-publishers)
8. [Error Handling](#error-handling)
9. [Backpressure](#backpressure)
10. [Threading and Schedulers](#threading-and-schedulers)
11. [Side Effects and Debugging](#side-effects-and-debugging)
12. [Testing Reactive Code](#testing-reactive-code)
13. [Real-World Patterns](#real-world-patterns)
14. [Best Practices](#best-practices)

---

## Reactor Fundamentals

### What is Project Reactor?

```java
/**
 * Project Reactor - Reactive Streams implementation
 * 
 * Core Types:
 * - Mono<T>  - 0 or 1 element (like CompletableFuture)
 * - Flux<T>  - 0 to N elements (like Stream)
 * 
 * Key Concepts:
 * - Non-blocking I/O
 * - Backpressure
 * - Asynchronous processing
 * - Declarative composition
 * 
 * Maven Dependency:
 * <dependency>
 *   <groupId>io.projectreactor</groupId>
 *   <artifactId>reactor-core</artifactId>
 *   <version>3.6.0</version>
 * </dependency>
 */

// Traditional blocking approach
List<User> users = userRepository.findAll();  // Blocks thread!
users.forEach(user -> sendEmail(user));       // Sequential

// Reactive approach
Flux<User> users = userRepository.findAll();  // Non-blocking publisher
users.flatMap(user -> sendEmail(user))        // Async, parallel
     .subscribe();                             // Subscribe to start
```

### Nothing Happens Until You Subscribe

```java
// ❌ No subscription - nothing happens!
Mono<String> mono = Mono.just("Hello");
mono.map(String::toUpperCase);  // Nothing executed!

// ✅ Subscribe to execute
Mono<String> mono = Mono.just("Hello");
mono.map(String::toUpperCase)
    .subscribe(System.out::println);  // Prints: HELLO

// Subscription variations
mono.subscribe();  // Fire and forget
mono.subscribe(System.out::println);  // Consumer
mono.subscribe(
    value -> System.out.println("Value: " + value),  // onNext
    error -> System.err.println("Error: " + error),  // onError
    () -> System.out.println("Completed")            // onComplete
);
```

---

## Mono\<T\>

### Creating Mono

```java
import reactor.core.publisher.Mono;

// From value
Mono<String> mono1 = Mono.just("Hello");

// Empty mono
Mono<String> mono2 = Mono.empty();

// From supplier (lazy)
Mono<String> mono3 = Mono.fromSupplier(() -> "Hello");

// From callable
Mono<String> mono4 = Mono.fromCallable(() -> expensiveOperation());

// From CompletableFuture
CompletableFuture<String> future = CompletableFuture.completedFuture("Hello");
Mono<String> mono5 = Mono.fromFuture(future);

// Error mono
Mono<String> mono6 = Mono.error(new RuntimeException("Error"));

// Defer creation (lazy)
Mono<String> mono7 = Mono.defer(() -> Mono.just(getValue()));

// Never completes (for testing)
Mono<String> mono8 = Mono.never();
```

### Transforming Mono

```java
Mono<String> mono = Mono.just("hello");

// map - transform value
Mono<String> upper = mono.map(String::toUpperCase);  // "HELLO"

// flatMap - transform to another Mono
Mono<Integer> length = mono.flatMap(s -> Mono.just(s.length()));

// filter - keep only if predicate true
Mono<String> filtered = mono.filter(s -> s.length() > 3);

// defaultIfEmpty - provide default for empty mono
Mono<String> withDefault = Mono.empty()
    .defaultIfEmpty("default");

// switchIfEmpty - switch to another mono if empty
Mono<String> switched = Mono.empty()
    .switchIfEmpty(Mono.just("fallback"));

// zipWith - combine with another mono
Mono<String> mono2 = Mono.just("world");
Mono<String> combined = mono.zipWith(mono2, (a, b) -> a + " " + b);
```

---

## Flux\<T\>

### Creating Flux

```java
import reactor.core.publisher.Flux;

// From values
Flux<String> flux1 = Flux.just("A", "B", "C");

// From array
String[] array = {"A", "B", "C"};
Flux<String> flux2 = Flux.fromArray(array);

// From iterable
List<String> list = Arrays.asList("A", "B", "C");
Flux<String> flux3 = Flux.fromIterable(list);

// From stream
Stream<String> stream = Stream.of("A", "B", "C");
Flux<String> flux4 = Flux.fromStream(stream);

// Range
Flux<Integer> flux5 = Flux.range(1, 10);  // 1 to 10

// Interval (time-based)
Flux<Long> flux6 = Flux.interval(Duration.ofSeconds(1));  // Emit every second

// Empty flux
Flux<String> flux7 = Flux.empty();

// Error flux
Flux<String> flux8 = Flux.error(new RuntimeException("Error"));

// Generate - programmatic creation
Flux<Integer> flux9 = Flux.generate(
    () -> 0,  // Initial state
    (state, sink) -> {
        sink.next(state);
        if (state == 10) sink.complete();
        return state + 1;
    }
);

// Create - for multiple emissions
Flux<String> flux10 = Flux.create(sink -> {
    sink.next("A");
    sink.next("B");
    sink.next("C");
    sink.complete();
});
```

### Transforming Flux

```java
Flux<Integer> flux = Flux.range(1, 10);

// map
Flux<Integer> doubled = flux.map(i -> i * 2);

// flatMap - async transformation
Flux<String> result = flux.flatMap(i -> 
    Mono.just("Number: " + i)
        .delayElement(Duration.ofMillis(100))
);

// filter
Flux<Integer> evens = flux.filter(i -> i % 2 == 0);

// take - first N elements
Flux<Integer> first3 = flux.take(3);

// skip - skip first N
Flux<Integer> afterFirst3 = flux.skip(3);

// distinct - remove duplicates
Flux<Integer> unique = Flux.just(1, 2, 2, 3, 3, 4).distinct();

// collectList - collect to Mono<List>
Mono<List<Integer>> list = flux.collectList();

// reduce - aggregate
Mono<Integer> sum = flux.reduce(0, Integer::sum);

// buffer - group into batches
Flux<List<Integer>> batches = flux.buffer(3);  // Groups of 3

// window - like buffer but returns Flux
Flux<Flux<Integer>> windows = flux.window(3);
```

---

## Creating Publishers

### From Blocking Code

```java
// ❌ Bad - blocking in reactive chain
Mono<User> user = Mono.just(
    userRepository.findById(id)  // Blocks!
);

// ✅ Good - defer blocking call
Mono<User> user = Mono.fromCallable(() -> 
    userRepository.findById(id)
).subscribeOn(Schedulers.boundedElastic());

// ✅ Better - use reactive repository
Mono<User> user = reactiveUserRepository.findById(id);
```

### From Callback APIs

```java
// Legacy callback API
public interface LegacyService {
    void getData(Callback callback);
}

// Wrap in Mono
public Mono<Data> getDataReactive() {
    return Mono.create(sink -> {
        legacyService.getData(new Callback() {
            @Override
            public void onSuccess(Data data) {
                sink.success(data);
            }
            
            @Override
            public void onError(Exception e) {
                sink.error(e);
            }
        });
    });
}
```

### From CompletableFuture

```java
// Convert CompletableFuture to Mono
CompletableFuture<String> future = asyncService.getData();
Mono<String> mono = Mono.fromFuture(future);

// Convert Mono to CompletableFuture
Mono<String> mono = Mono.just("Hello");
CompletableFuture<String> future = mono.toFuture();
```

---

## Transformation Operators

### map vs flatMap

```java
Flux<Integer> numbers = Flux.range(1, 5);

// map - synchronous transformation
Flux<String> strings = numbers.map(i -> "Number: " + i);

// flatMap - async transformation, preserves order
Flux<String> async = numbers.flatMap(i -> 
    Mono.just("Number: " + i)
        .delayElement(Duration.ofMillis(100))
);

// flatMapSequential - maintains order
Flux<String> ordered = numbers.flatMapSequential(i -> 
    Mono.just("Number: " + i)
        .delayElement(Duration.ofMillis(100))
);

// concatMap - sequential, one at a time
Flux<String> sequential = numbers.concatMap(i -> 
    Mono.just("Number: " + i)
        .delayElement(Duration.ofMillis(100))
);
```

### Real-World Example: User Processing

```java
public class UserService {
    
    // Process users in parallel
    public Flux<UserResult> processUsersParallel(Flux<User> users) {
        return users
            .flatMap(this::processUser);  // Parallel processing
    }
    
    // Process users maintaining order
    public Flux<UserResult> processUsersOrdered(Flux<User> users) {
        return users
            .flatMapSequential(this::processUser);  // Ordered
    }
    
    // Process users one at a time
    public Flux<UserResult> processUsersSequential(Flux<User> users) {
        return users
            .concatMap(this::processUser);  // Sequential
    }
    
    private Mono<UserResult> processUser(User user) {
        return Mono.fromCallable(() -> {
            // Expensive operation
            Thread.sleep(100);
            return new UserResult(user);
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

### collectList and collectMap

```java
Flux<User> users = userRepository.findAll();

// Collect to List
Mono<List<User>> userList = users.collectList();

// Collect to Map
Mono<Map<String, User>> userMap = users.collectMap(User::getId);

// Custom collection
Mono<Set<String>> names = users
    .map(User::getName)
    .collect(Collectors.toSet());

// GroupBy
Flux<GroupedFlux<String, User>> byCountry = users
    .groupBy(User::getCountry);

byCountry.flatMap(group -> 
    group.collectList()
        .map(list -> new CountryUsers(group.key(), list))
).subscribe();
```

---

## Filtering Operators

### Basic Filtering

```java
Flux<Integer> numbers = Flux.range(1, 100);

// filter
Flux<Integer> evens = numbers.filter(i -> i % 2 == 0);

// filterWhen - async filtering
Flux<User> activeUsers = users.filterWhen(user -> 
    checkIfActive(user)  // Returns Mono<Boolean>
);

// take - first N
Flux<Integer> first10 = numbers.take(10);

// takeLast - last N
Flux<Integer> last10 = numbers.takeLast(10);

// takeWhile - until predicate false
Flux<Integer> whileLessThan50 = numbers.takeWhile(i -> i < 50);

// takeUntil - until predicate true
Flux<Integer> untilGreaterThan50 = numbers.takeUntil(i -> i > 50);

// skip - skip first N
Flux<Integer> after10 = numbers.skip(10);

// skipLast - skip last N
Flux<Integer> exceptLast10 = numbers.skipLast(10);

// distinct
Flux<Integer> unique = Flux.just(1, 2, 2, 3, 3, 4).distinct();

// distinctUntilChanged - consecutive duplicates
Flux<Integer> deduped = Flux.just(1, 1, 2, 2, 3, 1, 1)
    .distinctUntilChanged();  // 1, 2, 3, 1
```

### Real-World: Pagination

```java
public class UserController {
    
    public Flux<User> getUsers(int page, int size) {
        return userRepository.findAll()
            .skip((long) page * size)
            .take(size);
    }
    
    public Flux<User> getUsersUntilInactive() {
        return userRepository.findAll()
            .takeWhile(User::isActive);
    }
}
```

---

## Combining Publishers

### merge and concat

```java
Flux<String> flux1 = Flux.just("A", "B", "C");
Flux<String> flux2 = Flux.just("D", "E", "F");

// merge - interleave (no order guarantee)
Flux<String> merged = Flux.merge(flux1, flux2);

// concat - sequential (first complete, then second)
Flux<String> concatenated = Flux.concat(flux1, flux2);

// zip - combine pairwise
Flux<String> zipped = Flux.zip(flux1, flux2, (a, b) -> a + b);

// combineLatest - emit when any changes
Flux<String> combined = Flux.combineLatest(
    flux1, flux2, 
    (a, b) -> a + b
);
```

### Real-World: Multiple Service Calls

```java
public class DataAggregator {
    
    // Call services in parallel, combine results
    public Mono<AggregatedData> aggregateData(String userId) {
        Mono<User> user = userService.getUser(userId);
        Mono<List<Order>> orders = orderService.getOrders(userId);
        Mono<Profile> profile = profileService.getProfile(userId);
        
        // Zip all together
        return Mono.zip(user, orders, profile)
            .map(tuple -> new AggregatedData(
                tuple.getT1(),  // user
                tuple.getT2(),  // orders
                tuple.getT3()   // profile
            ));
    }
    
    // Try multiple sources, use first success
    public Mono<Data> getDataWithFallback() {
        return Mono.first(
            primaryService.getData(),
            secondaryService.getData(),
            cacheService.getData()
        );
    }
}
```

---

## Error Handling

### onError Operators

```java
Mono<String> mono = Mono.error(new RuntimeException("Error!"));

// onErrorReturn - return default value
Mono<String> withDefault = mono.onErrorReturn("default");

// onErrorResume - switch to fallback mono
Mono<String> withFallback = mono.onErrorResume(error -> 
    Mono.just("fallback")
);

// onErrorResume with different handling per error type
Mono<String> handled = mono.onErrorResume(error -> {
    if (error instanceof TimeoutException) {
        return Mono.just("timeout");
    } else if (error instanceof IOException) {
        return Mono.just("io error");
    }
    return Mono.error(error);  // Re-throw
});

// onErrorMap - transform error
Mono<String> mapped = mono.onErrorMap(error -> 
    new CustomException("Wrapped", error)
);

// onErrorContinue - skip error, continue with next
Flux<Integer> flux = Flux.range(1, 5)
    .map(i -> {
        if (i == 3) throw new RuntimeException("Error at 3");
        return i;
    })
    .onErrorContinue((error, value) -> 
        System.err.println("Error at: " + value)
    );
// Emits: 1, 2, 4, 5 (skips 3)
```

### Retry Strategies

```java
Mono<String> mono = Mono.fromCallable(() -> {
    if (Math.random() < 0.8) {
        throw new RuntimeException("Random failure");
    }
    return "Success";
});

// Simple retry - N times
Mono<String> withRetry = mono.retry(3);

// Retry with predicate
Mono<String> retryPredicate = mono.retryWhen(
    Retry.fixedDelay(3, Duration.ofSeconds(1))
        .filter(error -> error instanceof TimeoutException)
);

// Exponential backoff
Mono<String> exponential = mono.retryWhen(
    Retry.backoff(3, Duration.ofSeconds(1))
        .maxBackoff(Duration.ofSeconds(10))
);

// Custom retry
Mono<String> customRetry = mono.retryWhen(
    Retry.fixedDelay(3, Duration.ofSeconds(2))
        .doBeforeRetry(signal -> 
            System.out.println("Retrying: " + signal.totalRetries())
        )
);
```

### Real-World: HTTP Client with Retry

```java
@Service
public class ResilientHttpClient {
    
    private final WebClient webClient;
    
    public Mono<String> callWithRetry(String url) {
        return webClient.get()
            .uri(url)
            .retrieve()
            .bodyToMono(String.class)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
                .filter(this::isRetryable)
                .onRetryExhaustedThrow((spec, signal) -> 
                    new ServiceUnavailableException("Service down")
                )
            )
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(TimeoutException.class, e -> 
                Mono.just("Timeout fallback")
            );
    }
    
    private boolean isRetryable(Throwable error) {
        return error instanceof WebClientResponseException.ServiceUnavailable ||
               error instanceof WebClientResponseException.GatewayTimeout;
    }
}
```

---

## Backpressure

### Understanding Backpressure

```java
/**
 * Backpressure - Consumer controls rate of production
 * 
 * Strategies:
 * - BUFFER - buffer all elements (risk OutOfMemory)
 * - DROP - drop oldest elements
 * - LATEST - keep only latest
 * - ERROR - throw error when overwhelmed
 */

// Fast producer, slow consumer
Flux.range(1, 1000)
    .delayElements(Duration.ofMillis(1))  // Fast
    .doOnNext(i -> System.out.println("Produced: " + i))
    .onBackpressureBuffer(100)  // Buffer up to 100
    .delayElements(Duration.ofMillis(10))  // Slow consumer
    .subscribe(i -> System.out.println("Consumed: " + i));

// Drop strategy
Flux.interval(Duration.ofMillis(1))
    .onBackpressureDrop(dropped -> 
        System.out.println("Dropped: " + dropped)
    )
    .delayElements(Duration.ofMillis(10))
    .subscribe();

// Latest strategy
Flux.interval(Duration.ofMillis(1))
    .onBackpressureLatest()
    .delayElements(Duration.ofMillis(10))
    .subscribe();

// Error strategy
Flux.range(1, 1000)
    .onBackpressureError()
    .delayElements(Duration.ofMillis(10))
    .subscribe();
```

### Controlling Request Rate

```java
// limitRate - request N at a time
Flux.range(1, 1000)
    .limitRate(10)  // Request 10 at a time
    .subscribe();

// Custom subscription
Flux.range(1, 100)
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(10);  // Request first 10
        }
        
        @Override
        protected void hookOnNext(Integer value) {
            System.out.println(value);
            if (value % 10 == 0) {
                request(10);  // Request next 10
            }
        }
    });
```

---

## Threading and Schedulers

### Schedulers

```java
/**
 * Scheduler Types:
 * 
 * Schedulers.immediate()       - Current thread
 * Schedulers.single()          - Single reusable thread
 * Schedulers.parallel()        - Fixed pool (CPU cores)
 * Schedulers.boundedElastic()  - Elastic pool (I/O operations)
 * Schedulers.fromExecutor()    - Custom executor
 */

// subscribeOn - where subscription happens
Mono.fromCallable(() -> {
    System.out.println("Subscription: " + Thread.currentThread().getName());
    return "Hello";
})
.subscribeOn(Schedulers.boundedElastic())
.subscribe();

// publishOn - downstream execution
Flux.range(1, 5)
    .map(i -> {
        System.out.println("Map1: " + Thread.currentThread().getName());
        return i * 2;
    })
    .publishOn(Schedulers.parallel())
    .map(i -> {
        System.out.println("Map2: " + Thread.currentThread().getName());
        return i + 1;
    })
    .subscribe();
```

### Real-World: Async I/O

```java
@Service
public class UserService {
    
    // Blocking database call - use boundedElastic
    public Mono<User> findUser(String id) {
        return Mono.fromCallable(() -> 
            jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userRowMapper,
                id
            )
        ).subscribeOn(Schedulers.boundedElastic());
    }
    
    // CPU-intensive - use parallel
    public Flux<ProcessedData> processData(Flux<Data> data) {
        return data
            .publishOn(Schedulers.parallel())
            .map(this::cpuIntensiveOperation);
    }
    
    // Mix I/O and CPU
    public Mono<Result> processWithMixedOperations(String id) {
        return Mono.fromCallable(() -> fetchFromDB(id))
            .subscribeOn(Schedulers.boundedElastic())  // I/O
            .publishOn(Schedulers.parallel())           // Switch to parallel
            .map(this::cpuIntensiveTransform)           // CPU work
            .publishOn(Schedulers.boundedElastic())     // Switch back
            .flatMap(this::saveToDatabase);             // I/O
    }
}
```

---

## Side Effects and Debugging

### doOn Hooks

```java
Flux.range(1, 5)
    .doOnSubscribe(sub -> System.out.println("Subscribed"))
    .doOnRequest(n -> System.out.println("Requested: " + n))
    .doOnNext(i -> System.out.println("Next: " + i))
    .doOnComplete(() -> System.out.println("Completed"))
    .doOnError(e -> System.err.println("Error: " + e))
    .doOnCancel(() -> System.out.println("Cancelled"))
    .doFinally(signal -> System.out.println("Finally: " + signal))
    .subscribe();
```

### Logging

```java
Flux.range(1, 5)
    .log()  // Log all signals
    .map(i -> i * 2)
    .log("AfterMap")  // Log with category
    .subscribe();

// Custom logger
Flux.range(1, 5)
    .log("MyFlux", Level.FINE, SignalType.ON_NEXT)
    .subscribe();
```

### Checkpoint for Debugging

```java
Flux.range(1, 5)
    .map(i -> i * 2)
    .checkpoint("After doubling")  // Debug marker
    .filter(i -> i > 5)
    .checkpoint("After filtering")
    .subscribe();
```

---

## Testing Reactive Code

### StepVerifier

```java
import reactor.test.StepVerifier;

@Test
void testMono() {
    Mono<String> mono = Mono.just("Hello");
    
    StepVerifier.create(mono)
        .expectNext("Hello")
        .verifyComplete();
}

@Test
void testFlux() {
    Flux<Integer> flux = Flux.range(1, 5);
    
    StepVerifier.create(flux)
        .expectNext(1, 2, 3, 4, 5)
        .verifyComplete();
}

@Test
void testError() {
    Mono<String> mono = Mono.error(new RuntimeException("Error"));
    
    StepVerifier.create(mono)
        .expectError(RuntimeException.class)
        .verify();
}

@Test
void testWithVirtualTime() {
    StepVerifier.withVirtualTime(() -> 
        Flux.interval(Duration.ofDays(1)).take(3)
    )
    .expectSubscription()
    .thenAwait(Duration.ofDays(1))
    .expectNext(0L)
    .thenAwait(Duration.ofDays(1))
    .expectNext(1L)
    .thenAwait(Duration.ofDays(1))
    .expectNext(2L)
    .verifyComplete();
}
```

---

## Real-World Patterns

### Repository Pattern

```java
public interface ReactiveUserRepository {
    Mono<User> findById(String id);
    Flux<User> findAll();
    Mono<User> save(User user);
    Mono<Void> deleteById(String id);
}

@Repository
public class UserRepositoryImpl implements ReactiveUserRepository {
    
    private final R2dbcEntityTemplate template;
    
    @Override
    public Mono<User> findById(String id) {
        return template.selectOne(
            Query.query(Criteria.where("id").is(id)),
            User.class
        );
    }
    
    @Override
    public Flux<User> findAll() {
        return template.select(User.class).all();
    }
    
    @Override
    public Mono<User> save(User user) {
        return template.insert(user);
    }
    
    @Override
    public Mono<Void> deleteById(String id) {
        return template.delete(
            Query.query(Criteria.where("id").is(id)),
            User.class
        ).then();
    }
}
```

### Service Layer

```java
@Service
public class UserService {
    
    private final ReactiveUserRepository userRepository;
    private final EmailService emailService;
    
    public Mono<User> createUser(CreateUserRequest request) {
        return Mono.just(request)
            .map(this::validateRequest)
            .flatMap(this::checkEmailUnique)
            .map(this::toUser)
            .flatMap(userRepository::save)
            .flatMap(this::sendWelcomeEmail);
    }
    
    private CreateUserRequest validateRequest(CreateUserRequest request) {
        if (request.getEmail() == null) {
            throw new ValidationException("Email required");
        }
        return request;
    }
    
    private Mono<CreateUserRequest> checkEmailUnique(CreateUserRequest request) {
        return userRepository.findByEmail(request.getEmail())
            .flatMap(existing -> Mono.error(
                new DuplicateEmailException("Email exists")
            ))
            .then(Mono.just(request));
    }
    
    private User toUser(CreateUserRequest request) {
        return User.builder()
            .id(UUID.randomUUID().toString())
            .email(request.getEmail())
            .name(request.getName())
            .build();
    }
    
    private Mono<User> sendWelcomeEmail(User user) {
        return emailService.sendWelcome(user)
            .then(Mono.just(user));
    }
}
```

### REST Controller

```java
@RestController
@RequestMapping("/users")
public class UserController {
    
    private final UserService userService;
    
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable String id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    @GetMapping
    public Flux<User> getAllUsers() {
        return userService.findAll();
    }
    
    @PostMapping
    public Mono<User> createUser(@RequestBody CreateUserRequest request) {
        return userService.createUser(request);
    }
    
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> deleteUser(@PathVariable String id) {
        return userService.deleteById(id)
            .map(v -> ResponseEntity.noContent().<Void>build())
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
}
```

---

## Best Practices

### ✅ DO

1. **Use reactive all the way**
   ```java
   // ✅ Fully reactive
   return userRepository.findById(id)
       .flatMap(orderRepository::findByUser);
   
   // ❌ Blocking in reactive chain
   return Mono.just(blockingRepository.findById(id));
   ```

2. **Subscribe on the right scheduler**
   ```java
   // I/O operations
   .subscribeOn(Schedulers.boundedElastic())
   
   // CPU-intensive
   .subscribeOn(Schedulers.parallel())
   ```

3. **Handle errors properly**
   ```java
   return service.call()
       .onErrorResume(error -> Mono.just(fallback))
       .retry(3);
   ```

4. **Use operators correctly**
   ```java
   // Async transformation
   .flatMap(user -> getOrders(user))
   
   // Sync transformation
   .map(User::getName)
   ```

5. **Test with StepVerifier**
6. **Use doOn hooks for side effects**
7. **Apply backpressure strategies**
8. **Chain operations declaratively**
9. **Avoid subscribe() in application code**
10. **Use checkpoint() for debugging**

### ❌ DON'T

1. **Don't block**
2. **Don't mix blocking and reactive**
3. **Don't forget to subscribe**
4. **Don't use subscribe() everywhere**
5. **Don't ignore backpressure**

---

## Conclusion

**Key Takeaways:**

1. **Mono<T>** - 0 or 1 element
2. **Flux<T>** - 0 to N elements
3. **Nothing happens until subscribe()**
4. **Use appropriate schedulers**
5. **Handle errors with onError operators**
6. **Apply backpressure strategies**
7. **Test with StepVerifier**
8. **Be fully reactive - avoid blocking**

**Remember**: Reactive programming is about composing asynchronous data streams. Use operators to build declarative pipelines, not imperative code!
