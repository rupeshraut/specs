# Effective Ways to Use Modern Java Concurrency

A comprehensive guide to mastering concurrency in modern Java (Java 8 through Java 21+).

---

## Table of Contents

1. [Concurrency Fundamentals](#concurrency-fundamentals)
2. [Thread Safety Basics](#thread-safety-basics)
3. [Locks and Synchronization](#locks-and-synchronization)
4. [Atomic Variables](#atomic-variables)
5. [Concurrent Collections](#concurrent-collections)
6. [Thread Pools and Executors](#thread-pools-and-executors)
7. [CompletableFuture (Async Programming)](#completablefuture-async-programming)
8. [Reactive Streams (Project Reactor)](#reactive-streams-project-reactor)
9. [Virtual Threads (Java 21+)](#virtual-threads-java-21)
10. [Structured Concurrency (Java 21+)](#structured-concurrency-java-21)
11. [Concurrency Patterns](#concurrency-patterns)
12. [Performance and Monitoring](#performance-and-monitoring)
13. [Common Pitfalls](#common-pitfalls)
14. [Decision Tree: Which Approach to Use](#decision-tree-which-approach-to-use)

---

## Concurrency Fundamentals

### Understanding Concurrency vs Parallelism

```java
/**
 * CONCURRENCY: Multiple tasks making progress (not necessarily simultaneously)
 * - Can run on single CPU through time-slicing
 * - About dealing with multiple things at once
 * 
 * PARALLELISM: Multiple tasks executing simultaneously
 * - Requires multiple CPUs/cores
 * - About doing multiple things at once
 * 
 * Modern applications need both!
 */

public class ConcurrencyVsParallelism {
    
    // Concurrent but not parallel - tasks interleaved on single thread
    public void concurrent() {
        CompletableFuture.supplyAsync(() -> task1())
            .thenApply(result -> task2(result));  // Sequential execution
    }
    
    // Parallel - tasks run simultaneously
    public void parallel() {
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> task1());
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> task2());
        
        CompletableFuture.allOf(future1, future2).join();  // Both run at same time
    }
}
```

### The Java Memory Model

```java
public class JavaMemoryModel {
    
    /**
     * Key Concepts:
     * 1. Visibility: Changes made by one thread may not be visible to others
     * 2. Ordering: Instructions may be reordered by compiler/CPU
     * 3. Atomicity: Operations may not be atomic even if they look simple
     */
    
    // ❌ NOT thread-safe - visibility problem
    private boolean flag = false;
    
    public void unsafeExample() {
        // Thread 1
        new Thread(() -> {
            while (!flag) {
                // May loop forever! flag update might not be visible
            }
            System.out.println("Flag is true");
        }).start();
        
        // Thread 2
        new Thread(() -> {
            flag = true;  // Change might not be visible to Thread 1
        }).start();
    }
    
    // ✅ Thread-safe - volatile ensures visibility
    private volatile boolean safeFlag = false;
    
    public void safeExample() {
        // Thread 1
        new Thread(() -> {
            while (!safeFlag) {
                // Will see the update
            }
            System.out.println("Flag is true");
        }).start();
        
        // Thread 2
        new Thread(() -> {
            safeFlag = true;  // Visible to all threads immediately
        }).start();
    }
    
    /**
     * volatile guarantees:
     * - Visibility: All threads see the latest value
     * - Ordering: Prevents instruction reordering
     * 
     * volatile does NOT guarantee:
     * - Atomicity of compound operations (i++, check-then-act)
     */
}
```

---

## Thread Safety Basics

### 1. **Immutability (Best Approach)**

```java
public final class ImmutableUser {
    
    private final String id;
    private final String name;
    private final List<String> roles;
    
    public ImmutableUser(String id, String name, List<String> roles) {
        this.id = id;
        this.name = name;
        this.roles = List.copyOf(roles);  // Defensive copy
    }
    
    // Only getters, no setters
    public String getId() { return id; }
    public String getName() { return name; }
    public List<String> getRoles() { return roles; }  // Returns immutable list
    
    // Return new instance for modifications
    public ImmutableUser withName(String newName) {
        return new ImmutableUser(this.id, newName, this.roles);
    }
}

// Using Java Records (Java 14+)
public record User(String id, String name, List<String> roles) {
    // Automatically immutable!
    public User {
        roles = List.copyOf(roles);  // Compact constructor for defensive copy
    }
}

/**
 * Why immutability is best for concurrency:
 * ✅ No synchronization needed
 * ✅ No race conditions possible
 * ✅ Can be freely shared between threads
 * ✅ Simpler reasoning about code
 */
```

### 2. **Thread Confinement**

```java
public class ThreadConfinement {
    
    // ThreadLocal - each thread gets its own copy
    private static final ThreadLocal<DateFormat> dateFormat = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
    
    public String formatDate(Date date) {
        // Each thread uses its own DateFormat instance
        // No synchronization needed!
        return dateFormat.get().format(date);
    }
    
    // Stack confinement - variables never escape thread
    public void stackConfinement() {
        List<String> localList = new ArrayList<>();
        localList.add("item");
        // localList never escapes this method
        // Completely thread-safe
    }
    
    // Ad-hoc confinement - discipline-based
    private final Map<String, User> users = new HashMap<>();  // Not thread-safe
    
    // ONLY accessed from single thread (enforced by design/documentation)
    public void singleThreadedAccess() {
        users.put("1", new User("1", "John"));
    }
}
```

### 3. **Volatile Variables**

```java
public class VolatileExample {
    
    // Use volatile for simple flags
    private volatile boolean shutdown = false;
    
    public void run() {
        while (!shutdown) {
            // Do work
        }
    }
    
    public void shutdown() {
        shutdown = true;  // Immediately visible to all threads
    }
    
    // ❌ WRONG - volatile doesn't make this atomic
    private volatile int counter = 0;
    
    public void incrementWrong() {
        counter++;  // NOT atomic! Read-modify-write
    }
    
    // ✅ RIGHT - use AtomicInteger
    private final AtomicInteger atomicCounter = new AtomicInteger(0);
    
    public void incrementRight() {
        atomicCounter.incrementAndGet();  // Atomic operation
    }
}
```

---

## Locks and Synchronization

### 1. **Synchronized Blocks**

```java
public class SynchronizedExamples {
    
    private final Object lock = new Object();
    private int count = 0;
    
    // Method-level synchronization
    public synchronized void incrementMethod() {
        count++;
    }
    
    // Block-level synchronization (preferred - smaller critical section)
    public void incrementBlock() {
        // Non-critical work here (not synchronized)
        
        synchronized (lock) {
            count++;  // Only this needs synchronization
        }
        
        // More non-critical work
    }
    
    // Multiple locks for fine-grained locking
    private final Object readLock = new Object();
    private final Object writeLock = new Object();
    private String data = "";
    
    public String read() {
        synchronized (readLock) {
            return data;
        }
    }
    
    public void write(String newData) {
        synchronized (writeLock) {
            data = newData;
        }
    }
    
    /**
     * synchronized guarantees:
     * ✅ Mutual exclusion (only one thread at a time)
     * ✅ Visibility (changes visible to next thread acquiring lock)
     * ✅ Atomicity (entire block is atomic)
     * 
     * Downsides:
     * ❌ Performance overhead
     * ❌ Potential for deadlocks
     * ❌ No timeout/try-lock capability
     * ❌ Unfair by default
     */
}
```

### 2. **ReentrantLock (Advanced Locking)**

```java
public class ReentrantLockExamples {
    
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;
    
    // Basic usage
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // MUST be in finally block
        }
    }
    
    // Try-lock with timeout
    public boolean tryIncrement(long timeout, TimeUnit unit) throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Couldn't acquire lock
    }
    
    // Interruptible locking
    public void incrementInterruptible() throws InterruptedException {
        lock.lockInterruptibly();  // Can be interrupted while waiting
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
    
    // Fair lock - prevents starvation
    private final ReentrantLock fairLock = new ReentrantLock(true);
    
    public void fairIncrement() {
        fairLock.lock();  // Threads acquire lock in order
        try {
            count++;
        } finally {
            fairLock.unlock();
        }
    }
    
    /**
     * When to use ReentrantLock over synchronized:
     * ✅ Need try-lock or timed lock
     * ✅ Need interruptible lock acquisition
     * ✅ Need fair lock ordering
     * ✅ Need to check if lock is held
     * ✅ Need hand-over-hand locking
     */
}
```

### 3. **ReadWriteLock**

```java
public class ReadWriteLockExample {
    
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    
    private final Map<String, String> cache = new HashMap<>();
    
    // Multiple readers can access simultaneously
    public String get(String key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    // Only one writer, blocks all readers
    public void put(String key, String value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    // Pattern: Check with read lock, upgrade to write if needed
    public String getOrCompute(String key, Supplier<String> supplier) {
        readLock.lock();
        try {
            String value = cache.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            readLock.unlock();
        }
        
        // Upgrade to write lock
        writeLock.lock();
        try {
            // Double-check (another thread might have computed it)
            String value = cache.get(key);
            if (value == null) {
                value = supplier.get();
                cache.put(key, value);
            }
            return value;
        } finally {
            writeLock.unlock();
        }
    }
    
    /**
     * Use ReadWriteLock when:
     * ✅ Read-heavy workload (many reads, few writes)
     * ✅ Read operations are expensive
     * ✅ Lock is held for significant time
     * 
     * Don't use when:
     * ❌ Write-heavy or balanced read/write
     * ❌ Very short critical sections
     * ❌ Many writers (consider StampedLock)
     */
}
```

### 4. **StampedLock (Optimistic Locking - Java 8+)**

```java
public class StampedLockExample {
    
    private final StampedLock sl = new StampedLock();
    private double x, y;
    
    // Optimistic read - no blocking!
    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();  // Non-blocking
        double currentX = x;
        double currentY = y;
        
        if (!sl.validate(stamp)) {  // Check if write happened
            // Fall back to read lock if validation failed
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
    
    // Write lock
    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }
    
    // Read lock (if optimistic read not suitable)
    public double getX() {
        long stamp = sl.readLock();
        try {
            return x;
        } finally {
            sl.unlockRead(stamp);
        }
    }
    
    // Upgrade from read to write lock
    public void moveIfAtOrigin(double newX, double newY) {
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {  // Upgrade successful
                    stamp = ws;
                    x = newX;
                    y = newY;
                    break;
                } else {  // Upgrade failed
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
    
    /**
     * StampedLock advantages:
     * ✅ Optimistic reads - very fast for read-heavy workloads
     * ✅ Better performance than ReadWriteLock in many cases
     * ✅ Can upgrade/downgrade locks
     * 
     * Cautions:
     * ⚠️  Not reentrant (unlike ReentrantLock)
     * ⚠️  No condition variables
     * ⚠️  More complex API
     */
}
```

---

## Atomic Variables

### 1. **Basic Atomic Operations**

```java
public class AtomicExamples {
    
    // Atomic primitives
    private final AtomicInteger counter = new AtomicInteger(0);
    private final AtomicLong longCounter = new AtomicLong(0L);
    private final AtomicBoolean flag = new AtomicBoolean(false);
    
    // Basic operations
    public void basicOperations() {
        counter.incrementAndGet();  // i++ atomically
        counter.getAndIncrement();  // i++ atomically, returns old value
        counter.addAndGet(5);       // i += 5 atomically
        counter.compareAndSet(0, 1); // CAS operation
    }
    
    // Complex atomic operation using updateAndGet
    public int incrementIfPositive() {
        return counter.updateAndGet(current -> 
            current > 0 ? current + 1 : current
        );
    }
    
    // Atomic reference
    private final AtomicReference<User> currentUser = new AtomicReference<>();
    
    public void atomicReference() {
        User user = new User("1", "John");
        currentUser.set(user);
        
        // Atomic update
        currentUser.updateAndGet(u -> 
            u != null ? new User(u.id(), "Jane") : null
        );
        
        // CAS - only update if current value matches
        currentUser.compareAndSet(user, new User("2", "Bob"));
    }
    
    /**
     * When to use Atomic classes:
     * ✅ Simple counters, flags, references
     * ✅ Lock-free algorithms
     * ✅ High contention scenarios
     * ✅ Single-variable updates
     * 
     * When NOT to use:
     * ❌ Multiple variables need to be updated atomically
     * ❌ Complex state transitions
     * ❌ Long-running operations in update function
     */
}
```

### 2. **Atomic Field Updaters**

```java
public class AtomicFieldUpdaterExample {
    
    // For updating volatile fields atomically without wrapper object
    private volatile int count;
    
    private static final AtomicIntegerFieldUpdater<AtomicFieldUpdaterExample> countUpdater =
        AtomicIntegerFieldUpdater.newUpdater(
            AtomicFieldUpdaterExample.class,
            "count"
        );
    
    public void increment() {
        countUpdater.incrementAndGet(this);
    }
    
    /**
     * Use Field Updaters when:
     * ✅ Memory footprint is critical (avoids AtomicInteger overhead)
     * ✅ Have many instances of the class
     * ✅ Only one or two fields need atomic updates
     */
}
```

### 3. **LongAdder and LongAccumulator**

```java
public class LongAdderExample {
    
    // Better than AtomicLong for high contention
    private final LongAdder counter = new LongAdder();
    
    public void increment() {
        counter.increment();  // Very fast under high contention
    }
    
    public long getCount() {
        return counter.sum();  // May not be exact at any given moment
    }
    
    // Custom accumulation function
    private final LongAccumulator max = new LongAccumulator(Long::max, Long.MIN_VALUE);
    
    public void updateMax(long value) {
        max.accumulate(value);
    }
    
    public long getMax() {
        return max.get();
    }
    
    /**
     * LongAdder performance characteristics:
     * ✅ Much faster than AtomicLong under high contention
     * ✅ Trades accuracy for speed (sum may be slightly stale)
     * 
     * Use when:
     * ✅ High-frequency updates from many threads
     * ✅ Exact real-time value not critical
     * ✅ Metrics, counters, statistics
     * 
     * Don't use when:
     * ❌ Need exact values for business logic
     * ❌ Low contention (AtomicLong is simpler)
     */
}
```

---

## Concurrent Collections

### 1. **ConcurrentHashMap**

```java
public class ConcurrentHashMapExamples {
    
    private final ConcurrentHashMap<String, User> users = new ConcurrentHashMap<>();
    
    // Thread-safe put/get
    public void basicOperations() {
        users.put("1", new User("1", "John"));
        User user = users.get("1");
    }
    
    // Atomic operations
    public void atomicOperations() {
        // Put if absent
        users.putIfAbsent("1", new User("1", "John"));
        
        // Remove only if value matches
        users.remove("1", new User("1", "John"));
        
        // Replace only if key exists
        users.replace("1", new User("1", "Jane"));
        
        // Replace only if old value matches
        users.replace("1", 
            new User("1", "John"), 
            new User("1", "Jane")
        );
    }
    
    // Atomic compute operations
    public void computeOperations() {
        // Compute new value if absent
        users.computeIfAbsent("1", id -> loadUserFromDB(id));
        
        // Compute new value if present
        users.computeIfPresent("1", (id, user) -> 
            new User(id, user.name().toUpperCase())
        );
        
        // Always compute (can remove by returning null)
        users.compute("1", (id, existingUser) -> {
            if (existingUser == null) {
                return loadUserFromDB(id);
            }
            return new User(id, existingUser.name() + " (updated)");
        });
        
        // Merge - combine old and new values
        users.merge("1", new User("1", "Default"), (existing, newUser) -> 
            new User(existing.id(), existing.name() + ", " + newUser.name())
        );
    }
    
    // Bulk operations (parallel)
    public void bulkOperations() {
        // forEach - parallel iteration
        users.forEach(10, (key, value) -> {
            System.out.println(key + ": " + value);
        });
        
        // search - returns first match
        User found = users.search(10, (key, value) -> 
            value.name().startsWith("J") ? value : null
        );
        
        // reduce - combine all values
        String allNames = users.reduce(10,
            (key, value) -> value.name(),
            (name1, name2) -> name1 + ", " + name2
        );
    }
    
    // Size and counting
    public void sizeOperations() {
        int size = users.size();  // May be approximate
        long exactSize = users.mappingCount();  // More accurate
        boolean isEmpty = users.isEmpty();
    }
    
    /**
     * ConcurrentHashMap features:
     * ✅ Lock-free reads
     * ✅ Fine-grained locking for writes
     * ✅ No fail-fast iterators (no ConcurrentModificationException)
     * ✅ Atomic compound operations
     * ✅ Parallel bulk operations
     * 
     * Don't use when:
     * ❌ Need strict happens-before ordering
     * ❌ Need exact size at all times
     */
}
```

### 2. **CopyOnWriteArrayList**

```java
public class CopyOnWriteExample {
    
    private final CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();
    
    // Add/remove - creates new array copy
    public void modify() {
        listeners.add("listener1");  // Expensive - copies entire array
        listeners.remove("listener1");
    }
    
    // Iteration - uses snapshot, no ConcurrentModificationException
    public void iterate() {
        for (String listener : listeners) {
            // Safe even if another thread modifies list
            notifyListener(listener);
        }
    }
    
    /**
     * Use CopyOnWriteArrayList when:
     * ✅ Read-heavy workload (iteration >> modification)
     * ✅ Small to medium-sized lists
     * ✅ Event listeners, observers
     * 
     * Don't use when:
     * ❌ Frequent modifications
     * ❌ Large lists (copy cost is high)
     * ❌ Memory-constrained environments
     */
}
```

### 3. **BlockingQueue Implementations**

```java
public class BlockingQueueExamples {
    
    // Array-backed, bounded queue
    private final BlockingQueue<Task> arrayQueue = new ArrayBlockingQueue<>(100);
    
    // Linked-node, optionally bounded
    private final BlockingQueue<Task> linkedQueue = new LinkedBlockingQueue<>(100);
    
    // Priority queue
    private final BlockingQueue<Task> priorityQueue = new PriorityBlockingQueue<>();
    
    // Synchronous handoff (no capacity)
    private final BlockingQueue<Task> synchronousQueue = new SynchronousQueue<>();
    
    // Delayed tasks
    private final BlockingQueue<DelayedTask> delayQueue = new DelayQueue<>();
    
    // Producer
    public void produce(Task task) throws InterruptedException {
        arrayQueue.put(task);  // Blocks if queue is full
        
        // Or non-blocking variants:
        boolean added = arrayQueue.offer(task);  // Returns false if full
        boolean addedWithTimeout = arrayQueue.offer(task, 1, TimeUnit.SECONDS);
    }
    
    // Consumer
    public void consume() throws InterruptedException {
        Task task = arrayQueue.take();  // Blocks if queue is empty
        
        // Or non-blocking variants:
        Task polled = arrayQueue.poll();  // Returns null if empty
        Task polledWithTimeout = arrayQueue.poll(1, TimeUnit.SECONDS);
    }
    
    /**
     * Queue Selection Guide:
     * 
     * ArrayBlockingQueue:
     * ✅ Bounded, fair ordering option
     * ✅ Better performance than LinkedBlockingQueue
     * ❌ Fixed capacity
     * 
     * LinkedBlockingQueue:
     * ✅ Optionally unbounded
     * ✅ Better throughput for high concurrency
     * ❌ Higher memory usage
     * 
     * PriorityBlockingQueue:
     * ✅ Elements ordered by priority
     * ✅ Unbounded
     * ❌ No ordering for equal priority
     * 
     * SynchronousQueue:
     * ✅ Direct handoff (producer waits for consumer)
     * ✅ Zero capacity
     * ✅ Good for thread-to-thread communication
     * 
     * DelayQueue:
     * ✅ Elements become available after delay
     * ✅ Good for scheduled tasks
     */
}
```

### 4. **ConcurrentLinkedQueue and Deque**

```java
public class ConcurrentQueueExamples {
    
    // Lock-free queue
    private final ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
    
    // Lock-free deque
    private final ConcurrentLinkedDeque<String> deque = new ConcurrentLinkedDeque<>();
    
    public void queueOperations() {
        queue.offer("item");  // Add to tail
        String item = queue.poll();  // Remove from head
        
        deque.offerFirst("item");  // Add to head
        deque.offerLast("item");   // Add to tail
        String first = deque.pollFirst();
        String last = deque.pollLast();
    }
    
    /**
     * Use when:
     * ✅ Need non-blocking operations
     * ✅ Don't need blocking take/put
     * ✅ High-performance, low-latency requirements
     */
}
```

---

## Thread Pools and Executors

### 1. **Executor Framework Basics**

```java
public class ExecutorBasics {
    
    // Fixed thread pool
    private final ExecutorService fixedPool = Executors.newFixedThreadPool(10);
    
    // Cached thread pool - creates threads as needed
    private final ExecutorService cachedPool = Executors.newCachedThreadPool();
    
    // Single thread executor - guarantees sequential execution
    private final ExecutorService singleThread = Executors.newSingleThreadExecutor();
    
    // Scheduled executor
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(5);
    
    // Submit tasks
    public void submitTasks() {
        // Runnable - no result
        fixedPool.submit(() -> System.out.println("Task"));
        
        // Callable - returns result
        Future<String> future = fixedPool.submit(() -> {
            return "Result";
        });
        
        // Get result (blocking)
        try {
            String result = future.get();  // Blocks until complete
            String resultWithTimeout = future.get(1, TimeUnit.SECONDS);
        } catch (Exception e) {
            // Handle timeout, interruption, execution exception
        }
    }
    
    // Scheduled tasks
    public void scheduledTasks() {
        // Run once after delay
        scheduler.schedule(() -> System.out.println("Delayed task"), 
            5, TimeUnit.SECONDS);
        
        // Run periodically at fixed rate
        scheduler.scheduleAtFixedRate(() -> System.out.println("Periodic task"),
            0,      // Initial delay
            10,     // Period
            TimeUnit.SECONDS
        );
        
        // Run periodically with fixed delay between executions
        scheduler.scheduleWithFixedDelay(() -> System.out.println("Fixed delay task"),
            0,      // Initial delay
            10,     // Delay after completion
            TimeUnit.SECONDS
        );
    }
    
    // Proper shutdown
    @PreDestroy
    public void shutdown() {
        shutdownExecutor(fixedPool);
        shutdownExecutor(cachedPool);
        shutdownExecutor(singleThread);
        shutdownExecutor(scheduler);
    }
    
    private void shutdownExecutor(ExecutorService executor) {
        executor.shutdown();  // Reject new tasks
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();  // Force shutdown
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    System.err.println("Executor did not terminate");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. **Custom Thread Pools**

```java
public class CustomThreadPools {
    
    // Fully customized thread pool
    private final ExecutorService customPool = new ThreadPoolExecutor(
        10,                      // Core pool size
        50,                      // Maximum pool size
        60L,                     // Keep-alive time
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),  // Work queue
        new ThreadFactoryBuilder()
            .setNameFormat("worker-%d")
            .setDaemon(false)
            .setPriority(Thread.NORM_PRIORITY)
            .setUncaughtExceptionHandler((t, e) -> 
                logger.error("Uncaught exception in thread " + t.getName(), e)
            )
            .build(),
        new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
    );
    
    /**
     * Thread Pool Sizing:
     * 
     * CPU-bound tasks:
     * threads = cores + 1
     * 
     * I/O-bound tasks:
     * threads = cores * (1 + wait_time / compute_time)
     * 
     * Example: 90% I/O wait time
     * threads = cores * (1 + 0.9 / 0.1) = cores * 10
     */
    
    private int calculatePoolSize() {
        int cores = Runtime.getRuntime().availableProcessors();
        // For I/O-heavy workload (e.g., database calls, HTTP requests)
        return cores * 10;
    }
    
    /**
     * Rejection Policies:
     * 
     * AbortPolicy (default):
     * - Throws RejectedExecutionException
     * - Use when you need to know about rejection
     * 
     * CallerRunsPolicy:
     * - Runs task in caller's thread
     * - Provides throttling mechanism
     * 
     * DiscardPolicy:
     * - Silently discards task
     * - Use for non-critical tasks
     * 
     * DiscardOldestPolicy:
     * - Discards oldest task in queue
     * - Use when newer tasks are more important
     */
}
```

### 3. **ForkJoinPool**

```java
public class ForkJoinExample {
    
    private final ForkJoinPool forkJoinPool = new ForkJoinPool();
    
    // Recursive task (returns result)
    static class SumTask extends RecursiveTask<Long> {
        private final long[] array;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 1000;
        
        SumTask(long[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected Long compute() {
            if (end - start <= THRESHOLD) {
                // Base case - compute directly
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += array[i];
                }
                return sum;
            } else {
                // Recursive case - split and fork
                int mid = (start + end) / 2;
                SumTask left = new SumTask(array, start, mid);
                SumTask right = new SumTask(array, mid, end);
                
                left.fork();  // Async execute left
                long rightResult = right.compute();  // Compute right in current thread
                long leftResult = left.join();  // Wait for left
                
                return leftResult + rightResult;
            }
        }
    }
    
    public long parallelSum(long[] array) {
        return forkJoinPool.invoke(new SumTask(array, 0, array.length));
    }
    
    /**
     * Use ForkJoinPool when:
     * ✅ Recursive divide-and-conquer algorithms
     * ✅ CPU-bound parallel computations
     * ✅ Work-stealing beneficial (load balancing)
     * 
     * ForkJoinPool is used by:
     * - Parallel streams
     * - CompletableFuture (default executor)
     */
}
```

---

## CompletableFuture (Async Programming)

```java
public class CompletableFutureQuickRef {
    
    // See separate CompletableFuture guide for comprehensive coverage
    
    // Basic async execution
    public CompletableFuture<String> basicAsync() {
        return CompletableFuture.supplyAsync(() -> "result");
    }
    
    // Chaining operations
    public CompletableFuture<Integer> chaining() {
        return CompletableFuture.supplyAsync(() -> "hello")
            .thenApply(String::length)
            .thenApply(len -> len * 2);
    }
    
    // Combining multiple futures
    public CompletableFuture<String> combining() {
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
        
        return f1.thenCombine(f2, (s1, s2) -> s1 + " " + s2);
    }
    
    // Error handling
    public CompletableFuture<String> errorHandling() {
        return CompletableFuture.supplyAsync(() -> {
            throw new RuntimeException("Error");
        }).exceptionally(ex -> "Fallback value");
    }
}
```

---

## Reactive Streams (Project Reactor)

### 1. **Flux and Mono Basics**

```java
@Service
public class ReactiveExamples {
    
    // Mono - 0 or 1 element
    public Mono<User> getUser(String id) {
        return Mono.fromCallable(() -> userRepository.findById(id))
            .subscribeOn(Schedulers.boundedElastic());  // Non-blocking I/O
    }
    
    // Flux - 0 to N elements
    public Flux<User> getAllUsers() {
        return Flux.fromIterable(userRepository.findAll())
            .subscribeOn(Schedulers.parallel());  // CPU-bound
    }
    
    // Transformation
    public Flux<UserDTO> transformUsers() {
        return getAllUsers()
            .map(this::toDTO)
            .filter(dto -> dto.isActive());
    }
    
    // Combining streams
    public Flux<OrderSummary> combineData() {
        return getOrders()
            .flatMap(order -> 
                Mono.zip(
                    getUser(order.getUserId()),
                    getPayment(order.getPaymentId())
                ).map(tuple -> new OrderSummary(order, tuple.getT1(), tuple.getT2()))
            );
    }
    
    // Error handling
    public Mono<User> withErrorHandling(String id) {
        return getUser(id)
            .onErrorResume(ex -> {
                logger.error("Failed to get user", ex);
                return Mono.just(User.guest());
            })
            .timeout(Duration.ofSeconds(5))
            .retry(3);
    }
    
    // Backpressure handling
    public Flux<Event> withBackpressure() {
        return eventStream()
            .onBackpressureBuffer(1000)  // Buffer up to 1000 items
            .onBackpressureDrop()        // Drop items if buffer full
            .onBackpressureLatest();     // Keep only latest
    }
    
    /**
     * Scheduler types:
     * 
     * Schedulers.immediate():
     * - Current thread
     * 
     * Schedulers.single():
     * - Single reusable thread
     * 
     * Schedulers.parallel():
     * - Fixed pool for CPU-bound work
     * 
     * Schedulers.boundedElastic():
     * - Growing pool for I/O-bound work
     * - Replaces elastic() in Reactor 3.4+
     */
}
```

### 2. **Reactive vs Imperative**

```java
public class ReactiveVsImperative {
    
    // ❌ Blocking imperative approach
    public List<UserDTO> imperativeApproach(List<String> userIds) {
        return userIds.stream()
            .map(id -> userRepository.findById(id))  // Blocking!
            .map(this::enrichUser)                   // Blocking!
            .map(this::toDTO)
            .collect(Collectors.toList());
    }
    
    // ✅ Non-blocking reactive approach
    public Flux<UserDTO> reactiveApproach(List<String> userIds) {
        return Flux.fromIterable(userIds)
            .flatMap(id -> userRepository.findByIdAsync(id))  // Non-blocking
            .flatMap(this::enrichUserAsync)                   // Non-blocking
            .map(this::toDTO);
    }
    
    /**
     * Use Reactive when:
     * ✅ High concurrency requirements
     * ✅ I/O-bound operations
     * ✅ Need backpressure handling
     * ✅ Streaming data
     * ✅ Microservices communication
     * 
     * Stick with imperative when:
     * ✅ Simple CRUD operations
     * ✅ CPU-bound work
     * ✅ Team not familiar with reactive
     * ✅ Simpler debugging needed
     */
}
```

---

## Virtual Threads (Java 21+)

### 1. **Virtual Thread Basics**

```java
public class VirtualThreadExamples {
    
    // Create virtual thread
    public void createVirtualThread() {
        Thread vThread = Thread.startVirtualThread(() -> {
            System.out.println("Running in virtual thread");
        });
    }
    
    // Virtual thread builder
    public void virtualThreadBuilder() throws InterruptedException {
        Thread vThread = Thread.ofVirtual()
            .name("virtual-worker")
            .unstarted(() -> System.out.println("Virtual thread"));
        
        vThread.start();
        vThread.join();
    }
    
    // Executor with virtual threads
    private final ExecutorService virtualExecutor = 
        Executors.newVirtualThreadPerTaskExecutor();
    
    public void useVirtualExecutor() {
        virtualExecutor.submit(() -> {
            // Each task gets its own virtual thread
            // Can handle millions of concurrent tasks!
            performBlockingOperation();
        });
    }
    
    // Real-world example: Handle many concurrent requests
    public void handleManyRequests() throws InterruptedException {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();
            
            // Submit 1 million tasks (!)
            for (int i = 0; i < 1_000_000; i++) {
                int taskId = i;
                futures.add(executor.submit(() -> {
                    // Blocking I/O is fine with virtual threads
                    return fetchDataFromAPI(taskId);
                }));
            }
            
            // Process results
            for (Future<String> future : futures) {
                String result = future.get();
                processResult(result);
            }
        }
    }
    
    /**
     * Virtual Threads characteristics:
     * ✅ Extremely lightweight (millions possible)
     * ✅ Perfect for I/O-bound tasks
     * ✅ Blocking code is OK (thread parks, not blocks)
     * ✅ Simpler than reactive programming
     * ✅ Works with existing thread-based APIs
     * 
     * When to use:
     * ✅ High concurrency I/O operations
     * ✅ Microservices with many blocking calls
     * ✅ Want simplicity of imperative code
     * 
     * Avoid for:
     * ❌ CPU-bound tasks (use platform threads/ForkJoinPool)
     * ❌ synchronized blocks (pins virtual thread)
     * ❌ Native code calls
     */
}
```

### 2. **Migrating to Virtual Threads**

```java
public class MigratingToVirtualThreads {
    
    // BEFORE: Traditional thread pool
    private final ExecutorService oldExecutor = 
        Executors.newFixedThreadPool(200);
    
    public void oldApproach() {
        oldExecutor.submit(() -> {
            // Limited to 200 concurrent tasks
            callExternalAPI();
        });
    }
    
    // AFTER: Virtual threads
    private final ExecutorService newExecutor = 
        Executors.newVirtualThreadPerTaskExecutor();
    
    public void newApproach() {
        newExecutor.submit(() -> {
            // Can handle millions of concurrent tasks
            callExternalAPI();
        });
    }
    
    // Spring Boot example
    @Configuration
    public class AsyncConfig {
        
        @Bean
        public AsyncTaskExecutor taskExecutor() {
            // Before Java 21
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(50);
            executor.setMaxPoolSize(200);
            return executor;
            
            // Java 21+ with virtual threads
            // return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
        }
    }
    
    /**
     * Migration checklist:
     * ✅ Replace fixed thread pools with virtual thread executor
     * ✅ Remove thread pool size tuning
     * ✅ Remove connection pool size limits (can be much higher)
     * ⚠️  Check for synchronized blocks (consider ReentrantLock)
     * ⚠️  Review ThreadLocal usage (still works but more instances)
     */
}
```

---

## Structured Concurrency (Java 21+)

### 1. **StructuredTaskScope Basics**

```java
public class StructuredConcurrencyExamples {
    
    // Shutdown on failure - all tasks canceled if any fails
    public OrderSummary getOrderSummaryShutdownOnFailure(String orderId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            // Start all tasks
            Supplier<Order> orderTask = scope.fork(() -> fetchOrder(orderId));
            Supplier<User> userTask = scope.fork(() -> fetchUser(orderId));
            Supplier<Payment> paymentTask = scope.fork(() -> fetchPayment(orderId));
            
            // Wait for all to complete (or first failure)
            scope.join()
                 .throwIfFailed();  // Throws if any task failed
            
            // All tasks succeeded - get results
            return new OrderSummary(
                orderTask.get(),
                userTask.get(),
                paymentTask.get()
            );
        }
    }
    
    // Shutdown on success - cancels other tasks when first succeeds
    public String getFastestResponse() throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            
            // Race multiple sources
            scope.fork(() -> fetchFromPrimaryAPI());
            scope.fork(() -> fetchFromSecondaryAPI());
            scope.fork(() -> fetchFromCache());
            
            // Wait for first success
            scope.join();
            
            // Return first successful result
            return scope.result();
        }
    }
    
    // Custom task scope
    public class AllOrNothingScope<T> extends StructuredTaskScope<T> {
        private final List<T> results = new ArrayList<>();
        private volatile boolean failed = false;
        
        @Override
        protected void handleComplete(Subtask<? extends T> subtask) {
            if (subtask.state() == Subtask.State.SUCCESS) {
                synchronized (results) {
                    results.add(subtask.get());
                }
            } else {
                failed = true;
                shutdown();  // Cancel all other tasks
            }
        }
        
        public List<T> results() throws Exception {
            if (failed) {
                throw new Exception("One or more tasks failed");
            }
            return List.copyOf(results);
        }
    }
    
    /**
     * Structured Concurrency benefits:
     * ✅ Automatic cleanup (try-with-resources)
     * ✅ Automatic cancellation propagation
     * ✅ Clear task hierarchy
     * ✅ No forgotten tasks
     * ✅ Better error handling
     * 
     * vs. CompletableFuture:
     * ✅ Simpler for parallel tasks with same lifetime
     * ✅ Better resource management
     * ❌ Less flexible for complex chains
     */
}
```

---

## Concurrency Patterns

### 1. **Producer-Consumer Pattern**

```java
public class ProducerConsumer {
    
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);
    
    // Producer
    public void produce(Task task) throws InterruptedException {
        queue.put(task);
    }
    
    // Consumer
    public void startConsumers(int numConsumers) {
        for (int i = 0; i < numConsumers; i++) {
            executorService.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Task task = queue.take();
                        processTask(task);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }
    
    private void processTask(Task task) {
        // Process task
    }
}
```

### 2. **Double-Checked Locking (Lazy Singleton)**

```java
public class Singleton {
    
    // ❌ WRONG - not thread-safe
    private static Singleton instance;
    
    public static Singleton getInstanceWrong() {
        if (instance == null) {
            instance = new Singleton();  // Race condition!
        }
        return instance;
    }
    
    // ✅ CORRECT - double-checked locking with volatile
    private static volatile Singleton instanceCorrect;
    
    public static Singleton getInstance() {
        if (instanceCorrect == null) {  // First check (no locking)
            synchronized (Singleton.class) {
                if (instanceCorrect == null) {  // Second check (with lock)
                    instanceCorrect = new Singleton();
                }
            }
        }
        return instanceCorrect;
    }
    
    // ✅ BETTER - initialization-on-demand holder
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstanceBetter() {
        return Holder.INSTANCE;  // Thread-safe, lazy, no locking
    }
    
    // ✅ BEST - use enum (if inheritance not needed)
    public enum SingletonEnum {
        INSTANCE;
        
        public void doSomething() {
            // Business logic
        }
    }
}
```

### 3. **Thread-Safe Lazy Initialization**

```java
public class LazyInitialization {
    
    // Using computeIfAbsent (ConcurrentHashMap)
    private final ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();
    
    public User getUser(String id) {
        return cache.computeIfAbsent(id, this::loadUser);
    }
    
    // Using Double-Checked Locking
    private volatile ExpensiveObject expensiveObject;
    
    public ExpensiveObject getExpensiveObject() {
        if (expensiveObject == null) {
            synchronized (this) {
                if (expensiveObject == null) {
                    expensiveObject = new ExpensiveObject();
                }
            }
        }
        return expensiveObject;
    }
}
```

### 4. **Future Timeout Pattern**

```java
public class TimeoutPatterns {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    // Using Future.get with timeout
    public String withFutureTimeout() throws Exception {
        Future<String> future = executor.submit(() -> slowOperation());
        
        try {
            return future.get(5, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);  // Cancel the task
            return "Operation timed out";
        }
    }
    
    // Using CompletableFuture.orTimeout (Java 9+)
    public CompletableFuture<String> withCompletableFutureTimeout() {
        return CompletableFuture.supplyAsync(() -> slowOperation())
            .orTimeout(5, TimeUnit.SECONDS)
            .exceptionally(ex -> "Operation timed out");
    }
}
```

### 5. **Barrier Pattern**

```java
public class BarrierPattern {
    
    private final CyclicBarrier barrier;
    private final List<Worker> workers;
    
    public BarrierPattern(int numWorkers) {
        // All threads wait at barrier, then action runs
        this.barrier = new CyclicBarrier(numWorkers, () -> {
            System.out.println("All workers completed, aggregating results");
        });
        
        this.workers = new ArrayList<>();
        for (int i = 0; i < numWorkers; i++) {
            workers.add(new Worker(barrier));
        }
    }
    
    static class Worker implements Runnable {
        private final CyclicBarrier barrier;
        
        Worker(CyclicBarrier barrier) {
            this.barrier = barrier;
        }
        
        @Override
        public void run() {
            try {
                // Do work
                System.out.println("Worker completed phase 1");
                barrier.await();  // Wait for others
                
                // Do more work
                System.out.println("Worker completed phase 2");
                barrier.await();  // Wait again
                
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    /**
     * Use CyclicBarrier when:
     * ✅ Fixed number of threads
     * ✅ Threads need to wait for each other
     * ✅ Reusable barrier (multiple rounds)
     * 
     * vs. CountDownLatch:
     * - CountDownLatch is one-time use
     * - CyclicBarrier can be reset
     */
}
```

### 6. **Semaphore for Resource Pooling**

```java
public class SemaphoreExample {
    
    private final Semaphore semaphore = new Semaphore(10);  // Max 10 concurrent
    
    public void accessResource() throws InterruptedException {
        semaphore.acquire();  // Get permit (blocks if none available)
        try {
            // Access limited resource (e.g., database connection)
            useResource();
        } finally {
            semaphore.release();  // Return permit
        }
    }
    
    // Try with timeout
    public boolean tryAccessResource(long timeout, TimeUnit unit) throws InterruptedException {
        if (semaphore.tryAcquire(timeout, unit)) {
            try {
                useResource();
                return true;
            } finally {
                semaphore.release();
            }
        }
        return false;  // Couldn't acquire permit
    }
    
    /**
     * Use Semaphore when:
     * ✅ Limiting concurrent access to resource
     * ✅ Implementing resource pools
     * ✅ Rate limiting
     */
}
```

---

## Performance and Monitoring

### 1. **Thread Monitoring**

```java
public class ThreadMonitoring {
    
    // Get thread information
    public void monitorThreads() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        
        // Thread count
        int threadCount = threadMXBean.getThreadCount();
        int peakThreadCount = threadMXBean.getPeakThreadCount();
        
        // Deadlock detection
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        if (deadlockedThreads != null) {
            System.out.println("Deadlock detected!");
            for (long threadId : deadlockedThreads) {
                ThreadInfo info = threadMXBean.getThreadInfo(threadId);
                System.out.println("Thread: " + info.getThreadName());
            }
        }
        
        // Thread CPU time
        long threadCpuTime = threadMXBean.getCurrentThreadCpuTime();
        long threadUserTime = threadMXBean.getCurrentThreadUserTime();
    }
    
    // Thread dump
    public void threadDump() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);
        
        for (ThreadInfo info : threadInfos) {
            System.out.println("Thread: " + info.getThreadName());
            System.out.println("State: " + info.getThreadState());
            
            StackTraceElement[] stackTrace = info.getStackTrace();
            for (StackTraceElement element : stackTrace) {
                System.out.println("  " + element);
            }
        }
    }
}
```

### 2. **Executor Service Monitoring**

```java
public class ExecutorMonitoring {
    
    private final ThreadPoolExecutor executor = (ThreadPoolExecutor) 
        Executors.newFixedThreadPool(10);
    
    public ExecutorStats getStats() {
        return new ExecutorStats(
            executor.getActiveCount(),      // Currently running tasks
            executor.getCompletedTaskCount(), // Total completed tasks
            executor.getTaskCount(),        // Total tasks (completed + queued + running)
            executor.getPoolSize(),         // Current pool size
            executor.getLargestPoolSize(),  // Peak pool size
            executor.getQueue().size()      // Queued tasks
        );
    }
    
    @Scheduled(fixedRate = 60000)  // Every minute
    public void logStats() {
        ExecutorStats stats = getStats();
        logger.info("Executor stats: active={}, completed={}, queued={}", 
            stats.active(), stats.completed(), stats.queued());
        
        // Alert if queue is growing
        if (stats.queued() > 1000) {
            alertService.sendAlert("Executor queue is large: " + stats.queued());
        }
    }
    
    record ExecutorStats(
        int active,
        long completed,
        long total,
        int poolSize,
        int peakPoolSize,
        int queued
    ) {}
}
```

### 3. **Performance Metrics**

```java
@Service
public class ConcurrencyMetrics {
    
    private final MeterRegistry meterRegistry;
    
    // Count active virtual threads
    private final AtomicInteger activeVirtualThreads = new AtomicInteger(0);
    
    public ConcurrencyMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Register gauges
        Gauge.builder("threads.active.virtual", activeVirtualThreads, AtomicInteger::get)
            .register(meterRegistry);
        
        Gauge.builder("threads.total", () -> 
            Thread.getAllStackTraces().keySet().size())
            .register(meterRegistry);
    }
    
    // Timer for async operations
    public CompletableFuture<String> timedAsyncOperation() {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        return CompletableFuture.supplyAsync(() -> {
            // Do work
            return "result";
        }).whenComplete((result, ex) -> {
            sample.stop(Timer.builder("async.operation")
                .tag("status", ex == null ? "success" : "failure")
                .register(meterRegistry));
        });
    }
}
```

---

## Common Pitfalls

### 1. **Race Conditions**

```java
public class RaceConditions {
    
    // ❌ WRONG - race condition
    private int counter = 0;
    
    public void incrementWrong() {
        counter++;  // Read-modify-write not atomic!
    }
    
    // ✅ CORRECT - synchronized
    public synchronized void incrementSync() {
        counter++;
    }
    
    // ✅ BETTER - atomic
    private final AtomicInteger atomicCounter = new AtomicInteger(0);
    
    public void incrementAtomic() {
        atomicCounter.incrementAndGet();
    }
    
    // ❌ WRONG - check-then-act race condition
    private final Map<String, User> cache = new HashMap<>();
    
    public void putIfAbsentWrong(String key, User user) {
        if (!cache.containsKey(key)) {  // Check
            cache.put(key, user);        // Act (race window!)
        }
    }
    
    // ✅ CORRECT - atomic check-then-act
    private final ConcurrentHashMap<String, User> concurrentCache = new ConcurrentHashMap<>();
    
    public void putIfAbsentCorrect(String key, User user) {
        concurrentCache.putIfAbsent(key, user);  // Atomic
    }
}
```

### 2. **Deadlocks**

```java
public class DeadlockExample {
    
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    // ❌ DEADLOCK - different lock ordering
    public void method1() {
        synchronized (lock1) {
            synchronized (lock2) {
                // Work
            }
        }
    }
    
    public void method2() {
        synchronized (lock2) {  // Different order!
            synchronized (lock1) {
                // Work - DEADLOCK!
            }
        }
    }
    
    // ✅ CORRECT - consistent lock ordering
    public void method1Correct() {
        synchronized (lock1) {
            synchronized (lock2) {
                // Work
            }
        }
    }
    
    public void method2Correct() {
        synchronized (lock1) {  // Same order
            synchronized (lock2) {
                // Work - no deadlock
            }
        }
    }
    
    /**
     * Deadlock prevention strategies:
     * ✅ Consistent lock ordering
     * ✅ Use timeout locks (tryLock)
     * ✅ Use single lock
     * ✅ Use lock-free algorithms
     */
}
```

### 3. **Thread Starvation**

```java
public class ThreadStarvation {
    
    // ❌ WRONG - unfair lock, high priority threads starve others
    private final ReentrantLock unfairLock = new ReentrantLock(false);
    
    // ✅ CORRECT - fair lock
    private final ReentrantLock fairLock = new ReentrantLock(true);
    
    /**
     * Prevent starvation:
     * ✅ Use fair locks
     * ✅ Proper thread pool sizing
     * ✅ Avoid infinite loops holding locks
     * ✅ Monitor thread metrics
     */
}
```

### 4. **Memory Leaks**

```java
public class MemoryLeaks {
    
    // ❌ WRONG - ThreadLocal not cleaned up
    private static final ThreadLocal<List<byte[]>> data = new ThreadLocal<>();
    
    public void leakyThreadLocal() {
        data.set(new ArrayList<>());
        data.get().add(new byte[1024 * 1024]);  // 1MB
        // data.remove() never called - memory leak with thread pools!
    }
    
    // ✅ CORRECT - clean up ThreadLocal
    public void safeThreadLocal() {
        try {
            data.set(new ArrayList<>());
            data.get().add(new byte[1024 * 1024]);
        } finally {
            data.remove();  // Clean up
        }
    }
    
    // ❌ WRONG - executor not shut down
    public void leakyExecutor() {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        executor.submit(() -> System.out.println("Task"));
        // executor.shutdown() never called - threads never die!
    }
}
```

---

## Decision Tree: Which Approach to Use

```
START: Need concurrency for what?

├─ I/O-bound operations (DB, HTTP, Files)
│  │
│  ├─ Java 21+? ──YES──► Virtual Threads (simplest)
│  │                     Use: Executors.newVirtualThreadPerTaskExecutor()
│  │
│  └─ Java 8-20? ──────► CompletableFuture + Custom Executor
│                        Or: Reactive Streams (Reactor/RxJava)
│
├─ CPU-bound parallel computations
│  │
│  ├─ Simple parallel? ─► Parallel Streams
│  │                     Use: stream().parallel()
│  │
│  ├─ Complex divide-and-conquer? ─► ForkJoinPool
│  │                                 Use: RecursiveTask/RecursiveAction
│  │
│  └─ Custom parallelism? ─► ExecutorService + CompletableFuture
│
├─ Shared mutable state
│  │
│  ├─ Simple counter/flag? ─► Atomic classes
│  │                          Use: AtomicInteger, AtomicBoolean
│  │
│  ├─ Collection? ─► Concurrent collections
│  │                Use: ConcurrentHashMap, CopyOnWriteArrayList
│  │
│  ├─ Complex state? ─► Synchronized or ReentrantLock
│  │                    Prefer: Immutability + copy-on-write
│  │
│  └─ Read-heavy? ─► ReadWriteLock or StampedLock
│
├─ Task scheduling
│  │
│  ├─ One-time delay? ─► ScheduledExecutorService.schedule()
│  │
│  ├─ Periodic? ─► scheduleAtFixedRate() or scheduleWithFixedDelay()
│  │
│  └─ Cron-like? ─► Spring @Scheduled or Quartz
│
└─ Coordination between threads
   │
   ├─ Producer-Consumer? ─► BlockingQueue
   │
   ├─ Wait for N threads? ─► CountDownLatch or CyclicBarrier
   │
   ├─ Limit concurrency? ─► Semaphore
   │
   └─ Phased computation? ─► Phaser
```

### Quick Reference Table

| Scenario | Java 21+ | Java 8-20 | Notes |
|----------|----------|-----------|-------|
| High-concurrency I/O | Virtual Threads | CompletableFuture | Virtual threads simpler |
| CPU-bound parallel | ForkJoinPool | Same | Use parallel streams for simple cases |
| Shared counter | AtomicInteger | Same | LongAdder for high contention |
| Shared collection | ConcurrentHashMap | Same | Never synchronize HashMap |
| Read-heavy data | StampedLock | Same | Optimistic reads very fast |
| Task queue | BlockingQueue | Same | Pick right queue type |
| Scheduled tasks | ScheduledExecutor | Same | Spring @Scheduled for cron |
| Async composition | Structured Concurrency | CompletableFuture | SC simpler for parallel tasks |

---

## Best Practices Summary

### ✅ DO

1. **Prefer immutability** - Thread-safe by default
2. **Use high-level abstractions** - Executors over raw threads
3. **Size thread pools correctly** - CPU vs I/O bound
4. **Use concurrent collections** - ConcurrentHashMap, not synchronized HashMap
5. **Clean up resources** - Shutdown executors, remove ThreadLocals
6. **Use try-with-resources** - StructuredTaskScope, AutoCloseable executors
7. **Monitor and measure** - Thread dumps, metrics, profiling
8. **Test for concurrency bugs** - Use tools like jcstress

### ❌ DON'T

1. **Don't use raw threads** - Use ExecutorService
2. **Don't ignore InterruptedException** - Restore interrupt status
3. **Don't synchronize on public objects** - Use private locks
4. **Don't forget to shutdown executors** - Memory leaks
5. **Don't block with locks held** - Minimize critical sections
6. **Don't use synchronized HashMap** - Use ConcurrentHashMap
7. **Don't swallow exceptions** - Log and handle properly
8. **Don't assume atomicity** - i++ is not atomic

---

## Conclusion

Modern Java concurrency has evolved significantly:

**Java 8-11**: Streams, CompletableFuture, improved concurrent collections  
**Java 12-16**: StampedLock optimizations, pattern matching helps concurrent code  
**Java 17**: Sealed classes help with concurrent state machines  
**Java 19-20**: Virtual threads preview  
**Java 21+**: Virtual threads, structured concurrency - game changers!

**The Future**: Virtual threads make concurrent programming much simpler. Eventually, most I/O code will just use virtual threads instead of reactive programming or complex async code.

**Choose based on your needs**:
- **Simple & modern**: Virtual Threads (Java 21+)
- **Complex async**: CompletableFuture or Reactor
- **CPU-bound**: Parallel streams or ForkJoinPool
- **Shared state**: Concurrent collections + atomic classes
