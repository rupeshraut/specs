# 🧵 Java Concurrency & Virtual Threads Cheat Sheet

> **Purpose:** Production patterns for concurrent programming with virtual threads, structured concurrency, and thread-safe design. Reference before writing any concurrent code or parallelizing I/O.
> **Stack context:** Java 21+ / Spring Boot 3.x / Virtual Threads / CompletableFuture

---

## 📋 Concurrency Decision Framework

| Question | Answer |
|----------|--------|
| Is the task **I/O-bound**? (HTTP, DB, queue) | → Virtual threads |
| Is the task **CPU-bound**? (hashing, serialization) | → Platform thread pool (ForkJoinPool) |
| Do I need **parallel fan-out** then join? | → Structured Concurrency or CompletableFuture.allOf |
| Must I **limit concurrency**? (connection pool, rate limit) | → Semaphore or Bulkhead |
| Is **shared mutable state** involved? | → Eliminate it, or use lock/atomic/concurrent collection |
| Can the result be **cached**? | → ConcurrentHashMap.computeIfAbsent |

---

## ⚡ Pattern 1: Virtual Threads

### Virtual vs Platform Threads

```
PLATFORM THREADS (Traditional)
  • 1:1 mapping to OS thread
  • ~1MB stack each
  • Max practical: ~5,000 per JVM
  • Expensive to create/destroy
  • BLOCKS the OS thread on I/O

VIRTUAL THREADS (Java 21+)
  • Many-to-few mapping (M:N) to carrier (platform) threads
  • ~1KB initial stack (grows as needed)
  • Millions possible per JVM
  • Cheap to create/destroy
  • UNMOUNTS from carrier on I/O (carrier is freed)

WHEN TO USE VIRTUAL THREADS:
  ✅ HTTP request handling (Spring Boot server)
  ✅ Database calls
  ✅ REST client calls
  ✅ Kafka produce/consume
  ✅ JMS messaging
  ✅ File I/O
  ✅ Any blocking I/O operation

WHEN TO AVOID:
  ❌ CPU-intensive computation (no benefit — still needs CPU time)
  ❌ Code using synchronized (pins the carrier thread)
  ❌ Code relying on ThreadLocal with large objects (memory per VT)
```

### Creating Virtual Threads

```java
// ── Simple task ──
Thread.startVirtualThread(() -> {
    var result = httpClient.send(request, BodyHandlers.ofString());
    process(result);
});

// ── Executor (for structured work) ──
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> callServiceA());
    executor.submit(() -> callServiceB());
}  // Waits for all tasks to complete

// ── Named threads (for debugging) ──
var factory = Thread.ofVirtual().name("payment-worker-", 0).factory();
try (var executor = Executors.newThreadPerTaskExecutor(factory)) {
    executor.submit(() -> processPayment(payment));
}

// ── Spring Boot: enable globally ──
// application.yml:
// spring.threads.virtual.enabled: true
// All HTTP handling, @Async, Kafka consumers use virtual threads automatically
```

### Virtual Thread Pitfalls & Fixes

```java
// ═══ PITFALL 1: synchronized PINS the carrier thread ═══
// ❌ BAD: virtual thread is pinned — carrier thread blocked
public synchronized void process(Payment payment) {
    // This I/O call blocks the CARRIER thread, not just the VT
    var result = gateway.charge(payment);
}

// ✅ FIX: Use ReentrantLock (VT unmounts while waiting for lock)
private final ReentrantLock lock = new ReentrantLock();
public void process(Payment payment) {
    lock.lock();
    try {
        var result = gateway.charge(payment);
    } finally {
        lock.unlock();
    }
}


// ═══ PITFALL 2: ThreadLocal with heavy objects ═══
// ❌ BAD: one instance per virtual thread = millions of heavy objects
private static final ThreadLocal<ObjectMapper> MAPPER = 
    ThreadLocal.withInitial(ObjectMapper::new);

// ✅ FIX: Share immutable instance (ObjectMapper is thread-safe if not reconfigured)
private static final ObjectMapper MAPPER = new ObjectMapper()
    .registerModule(new JavaTimeModule());

// ✅ FIX: For truly thread-local needs, use ScopedValue (Java 24 preview)
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.runWhere(CONTEXT, new RequestContext(traceId), () -> {
    // CONTEXT.get() available in this scope and all child virtual threads
    processPayment(payment);
});


// ═══ PITFALL 3: Unbounded virtual thread creation ═══
// ❌ BAD: millions of VTs hitting a database with 50 connections
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var payment : millionsOfPayments) {
        executor.submit(() -> repository.save(payment));  // Connection pool exhaustion!
    }
}

// ✅ FIX: Use Semaphore to limit concurrency
private final Semaphore dbSemaphore = new Semaphore(40);  // Match pool size

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var payment : millionsOfPayments) {
        executor.submit(() -> {
            dbSemaphore.acquire();
            try {
                repository.save(payment);
            } finally {
                dbSemaphore.release();
            }
        });
    }
}


// ═══ PITFALL 4: parallelStream() ═══
// ❌ BAD: parallelStream uses ForkJoinPool, NOT virtual threads
payments.parallelStream().forEach(p -> gateway.charge(p));

// ✅ FIX: Explicit virtual thread executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = payments.stream()
        .map(p -> executor.submit(() -> gateway.charge(p)))
        .toList();
    futures.forEach(f -> { try { f.get(); } catch (Exception e) { handleError(e); }});
}
```

---

## 🏗️ Pattern 2: Structured Concurrency (Java 24 Preview)

> **Core idea:** Child tasks have the same lifetime as their parent scope. If the parent is cancelled, all children are cancelled. No orphaned tasks.

```java
// ── Fan-out: call multiple services in parallel, wait for all ──
public PaymentEnrichment enrichPayment(Payment payment) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

        // Launch concurrent tasks
        Subtask<FraudResult> fraudTask = scope.fork(() ->
            fraudService.evaluate(payment));
        Subtask<CustomerDetail> customerTask = scope.fork(() ->
            customerService.getDetail(payment.customerId()));
        Subtask<ExchangeRate> rateTask = scope.fork(() ->
            rateService.getRate(payment.currency(), "USD"));

        // Wait for ALL to complete (or first failure)
        scope.join();
        scope.throwIfFailed();   // Propagates first exception

        // All succeeded — collect results
        return new PaymentEnrichment(
            fraudTask.get(),
            customerTask.get(),
            rateTask.get()
        );
    }
    // If ANY task fails: all others are automatically cancelled
    // If parent thread is interrupted: all children are cancelled
    // No orphaned threads possible
}

// ── First-success: return first result, cancel the rest ──
public ChargeResult chargeWithFallback(Payment payment) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<ChargeResult>()) {

        scope.fork(() -> primaryGateway.charge(payment));
        scope.fork(() -> fallbackGateway.charge(payment));

        scope.join();
        return scope.result();   // First successful result
    }
    // Second gateway is cancelled once first succeeds
}
```

### Structured Concurrency vs CompletableFuture

```
STRUCTURED CONCURRENCY                    COMPLETABLE FUTURE
──────────────────────                    ───────────────────
Parent-child lifecycle bound              Fire-and-forget possible
Automatic cancellation propagation        Manual cancellation needed
No orphaned tasks possible                Orphaned futures = resource leak
Works naturally with virtual threads      Works with any executor
Clear ownership and error handling        .exceptionally() chains get complex
Java 24 preview                           Stable since Java 8

USE Structured Concurrency for:
  • Parallel I/O fan-out within a request
  • Any case where all tasks share a lifetime

USE CompletableFuture for:
  • Complex async pipelines with transformations
  • Interop with libraries returning CompletableFuture
  • Pre-Java 24 codebases
```

---

## 🔗 Pattern 3: CompletableFuture Composition

### Common Patterns

```java
// ── Parallel fan-out, combine results ──
public PaymentEnrichment enrichPayment(Payment payment) {
    var fraudFuture = CompletableFuture.supplyAsync(
        () -> fraudService.evaluate(payment), virtualExecutor);
    var customerFuture = CompletableFuture.supplyAsync(
        () -> customerService.getDetail(payment.customerId()), virtualExecutor);
    var rateFuture = CompletableFuture.supplyAsync(
        () -> rateService.getRate(payment.currency(), "USD"), virtualExecutor);

    return CompletableFuture.allOf(fraudFuture, customerFuture, rateFuture)
        .thenApply(v -> new PaymentEnrichment(
            fraudFuture.join(), customerFuture.join(), rateFuture.join()))
        .orTimeout(10, TimeUnit.SECONDS)
        .join();
}

// ── Sequential chain with transformation ──
CompletableFuture.supplyAsync(() -> findPayment(id), virtualExecutor)
    .thenApply(payment -> enrichWithFraud(payment))
    .thenApply(enriched -> calculateFees(enriched))
    .thenCompose(withFees -> chargeGatewayAsync(withFees))  // Returns another future
    .thenAccept(result -> publishEvent(result))
    .exceptionally(ex -> {
        log.error("Payment pipeline failed", ex);
        return null;
    });

// ── First to complete (racing) ──
var primary = CompletableFuture.supplyAsync(() -> primaryGateway.charge(payment));
var fallback = CompletableFuture.supplyAsync(() -> fallbackGateway.charge(payment));
var result = CompletableFuture.anyOf(primary, fallback)
    .orTimeout(5, TimeUnit.SECONDS)
    .join();

// ── Timeout with fallback ──
CompletableFuture.supplyAsync(() -> enrichmentService.enrich(payment))
    .completeOnTimeout(PaymentEnrichment.empty(), 3, TimeUnit.SECONDS)
    .thenApply(enrichment -> processWithEnrichment(payment, enrichment));

// ── Handle both success and failure ──
CompletableFuture.supplyAsync(() -> gateway.charge(payment))
    .handle((result, ex) -> {
        if (ex != null) {
            log.error("Charge failed: {}", ex.getMessage());
            return ChargeResult.failed(ex.getMessage());
        }
        return result;
    });
```

### CompletableFuture + Virtual Threads

```java
// Use virtual thread executor for I/O-bound CompletableFuture chains
private static final ExecutorService VIRTUAL_EXECUTOR =
    Executors.newVirtualThreadPerTaskExecutor();

// Every supplyAsync/runAsync uses virtual threads
CompletableFuture.supplyAsync(() -> blockingIoCall(), VIRTUAL_EXECUTOR);

// ⚠️ Without specifying executor, uses ForkJoinPool.commonPool()
// which has limited platform threads — BAD for blocking I/O
CompletableFuture.supplyAsync(() -> blockingIoCall());  // ❌ Uses common pool
```

---

## 🔒 Pattern 4: Thread-Safe Data Structures

### Choosing the Right Concurrent Collection

| Need | Use | Don't Use |
|------|-----|-----------|
| Key-value, high concurrency | `ConcurrentHashMap` | `Collections.synchronizedMap` |
| Set, high concurrency | `ConcurrentHashMap.newKeySet()` | `Collections.synchronizedSet` |
| Queue, producer-consumer | `LinkedBlockingQueue` | `ArrayList` with sync |
| Queue, bounded + backpressure | `ArrayBlockingQueue(capacity)` | Unbounded queue |
| Queue, many producers, one consumer | `ConcurrentLinkedQueue` | |
| List, mostly reads | `CopyOnWriteArrayList` | `synchronizedList` |
| Counter | `AtomicLong` / `LongAdder` | `synchronized int` |
| Accumulator | `LongAdder` (high contention) | `AtomicLong` |
| Reference swap | `AtomicReference<T>` | `volatile T` with race |

### Atomic Operations Pattern

```java
// ── AtomicReference for lock-free state machine ──
private final AtomicReference<ServiceState> state =
    new AtomicReference<>(ServiceState.STARTING);

public boolean transitionTo(ServiceState expected, ServiceState newState) {
    return state.compareAndSet(expected, newState);
}

// ── ConcurrentHashMap.computeIfAbsent for caching ──
private final ConcurrentHashMap<String, PaymentProcessor> processors = new ConcurrentHashMap<>();

public PaymentProcessor getProcessor(String type) {
    return processors.computeIfAbsent(type, PaymentProcessor::forType);
    // Thread-safe, computed only once per key
}

// ── LongAdder for high-contention counters ──
private final LongAdder requestCount = new LongAdder();

public void onRequest() {
    requestCount.increment();   // Much faster than AtomicLong under contention
}

public long getRequestCount() {
    return requestCount.sum();
}
```

---

## 🚧 Pattern 5: Synchronization Primitives

### Lock Selection Guide

```
ReentrantLock
  ✅ Virtual thread friendly (VT unmounts while waiting)
  ✅ tryLock() with timeout
  ✅ Fair ordering option
  Use: General-purpose mutual exclusion

ReadWriteLock (ReentrantReadWriteLock)
  ✅ Multiple concurrent readers, exclusive writer
  Use: Read-heavy caches, configuration that rarely changes

StampedLock
  ✅ Optimistic read (no locking overhead for reads if no contention)
  ⚠️ Non-reentrant, more complex API
  Use: Extremely read-heavy, performance-critical

Semaphore
  ✅ Controls concurrency level (N permits)
  ✅ Virtual thread friendly
  Use: Rate limiting, connection pool proxy, bounded parallelism

CountDownLatch
  ✅ One-time gate: wait for N events then proceed
  Use: Waiting for initialization, test synchronization

CyclicBarrier
  ✅ Reusable: N threads wait for each other, then all proceed
  Use: Phased computation
```

### Semaphore for Bounded Concurrency

```java
@Component
public class BoundedGatewayClient {

    private final Semaphore gatewayPermits;
    private final PaymentGateway gateway;

    public BoundedGatewayClient(
            PaymentGateway gateway,
            @Value("${gateway.max-concurrent:20}") int maxConcurrent) {
        this.gateway = gateway;
        this.gatewayPermits = new Semaphore(maxConcurrent);
    }

    public ChargeResult charge(Payment payment) {
        try {
            if (!gatewayPermits.tryAcquire(5, TimeUnit.SECONDS)) {
                throw new ServiceOverloadedException("Gateway at capacity");
            }
            try {
                return gateway.charge(payment);
            } finally {
                gatewayPermits.release();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ServiceException("Interrupted waiting for gateway permit", e);
        }
    }
}
```

---

## 🔄 Pattern 6: Producer-Consumer

```java
// ── Bounded queue with backpressure ──
@Component
public class PaymentProcessingPipeline {

    private final BlockingQueue<Payment> queue;
    private final ExecutorService workers;

    public PaymentProcessingPipeline(
            @Value("${pipeline.queue-capacity:1000}") int capacity,
            @Value("${pipeline.workers:10}") int workerCount) {
        this.queue = new ArrayBlockingQueue<>(capacity);  // Bounded!
        this.workers = Executors.newVirtualThreadPerTaskExecutor();

        // Start worker virtual threads
        for (int i = 0; i < workerCount; i++) {
            workers.submit(this::workerLoop);
        }
    }

    // Producer — blocks if queue is full (backpressure)
    public boolean submit(Payment payment) {
        try {
            return queue.offer(payment, 5, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }

    // Consumer worker
    private void workerLoop() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                var payment = queue.poll(1, TimeUnit.SECONDS);
                if (payment != null) {
                    processPayment(payment);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                log.error("Worker error", e);
            }
        }
    }

    @PreDestroy
    void shutdown() {
        workers.shutdown();
        try {
            if (!workers.awaitTermination(30, TimeUnit.SECONDS)) {
                workers.shutdownNow();
            }
        } catch (InterruptedException e) {
            workers.shutdownNow();
        }
    }
}
```

---

## 🚫 Concurrency Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **synchronized with virtual threads** | Pins carrier thread, kills VT benefit | ReentrantLock |
| **ThreadLocal with virtual threads** | Millions of copies, memory explosion | ScopedValue or pass as parameter |
| **parallelStream() for I/O** | Common pool has few threads, blocks them all | Virtual thread executor |
| **Unbounded thread creation** | Without concurrency limits, exhausts downstream | Semaphore, bounded executor |
| **catch (InterruptedException) { }** | Swallows interrupt, thread never stops | Re-interrupt: `Thread.currentThread().interrupt()` |
| **Double-checked locking (broken)** | Still fails without volatile | Use enum, Holder pattern, or AtomicReference |
| **Mutable shared state without sync** | Race conditions, data corruption | Immutable objects, concurrent collections |
| **sleep() for coordination** | Slow, unreliable | CountDownLatch, Semaphore, Awaitility in tests |
| **Future.get() without timeout** | Blocks forever on failure | Always `.get(timeout, unit)` or `.orTimeout()` |
| **Ignoring executor shutdown** | Orphaned threads, resource leak | try-with-resources or @PreDestroy shutdown |

---

## 💡 Golden Rules of Java Concurrency

```
1.  VIRTUAL THREADS for I/O — one line in application.yml, dramatic improvement.
2.  IMMUTABILITY is the best synchronization — records, final fields, unmodifiable collections.
3.  ReentrantLock > synchronized — always, especially with virtual threads.
4.  BOUND your concurrency — Semaphore to match downstream capacity (DB pool, gateway limit).
5.  STRUCTURED CONCURRENCY when available — no orphaned tasks, automatic cancellation.
6.  CompletableFuture + virtual executor — best combo for pre-Java 24 parallel I/O.
7.  NEVER swallow InterruptedException — re-interrupt the thread.
8.  TIMEOUT on every blocking call — .get(5, SECONDS), not .get().
9.  CONCURRENT collections > synchronized wrappers — ConcurrentHashMap, not synchronizedMap.
10. If concurrency is hard, ELIMINATE shared state — the best lock is the one you don't need.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Virtual Threads / Spring Boot 3.x*
