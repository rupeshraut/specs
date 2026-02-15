# Effective Modern Java Concurrency Utilities

A comprehensive guide to mastering Java concurrency utilities including synchronizers, locks, and coordination primitives.

---

## Table of Contents

1. [Concurrency Utilities Overview](#concurrency-utilities-overview)
2. [CountDownLatch](#countdownlatch)
3. [CyclicBarrier](#cyclicbarrier)
4. [Semaphore](#semaphore)
5. [Phaser](#phaser)
6. [Exchanger](#exchanger)
7. [ReentrantLock](#reentrantlock)
8. [ReadWriteLock](#readwritelock)
9. [StampedLock](#stampedlock)
10. [CompletableFuture](#completablefuture)
11. [ForkJoinPool](#forkjoinpool)
12. [Concurrent Collections](#concurrent-collections)
13. [Atomic Variables](#atomic-variables)
14. [Best Practices](#best-practices)
15. [Common Pitfalls](#common-pitfalls)

---

## Concurrency Utilities Overview

### Java Concurrency Package

```java
/**
 * java.util.concurrent provides:
 * 
 * Synchronizers:
 * - CountDownLatch     - Wait for set of operations to complete
 * - CyclicBarrier      - Barrier that threads wait at (reusable)
 * - Semaphore          - Control access to resource pool
 * - Phaser             - Flexible barrier with phases
 * - Exchanger          - Exchange data between two threads
 * 
 * Locks:
 * - ReentrantLock      - Explicit lock with more features than synchronized
 * - ReadWriteLock      - Separate read/write locks
 * - StampedLock        - Optimistic read lock (Java 8+)
 * 
 * Executors:
 * - ExecutorService    - Thread pool management
 * - ForkJoinPool       - Work-stealing thread pool
 * 
 * Concurrent Collections:
 * - ConcurrentHashMap  - Thread-safe HashMap
 * - CopyOnWriteArrayList - Thread-safe ArrayList
 * - BlockingQueue      - Producer-consumer queues
 * 
 * Atomic Variables:
 * - AtomicInteger, AtomicLong, AtomicReference, etc.
 */
```

---

## CountDownLatch

### Basic Usage

```java
import java.util.concurrent.CountDownLatch;

/**
 * CountDownLatch - One-shot barrier
 * 
 * Use when: Need to wait for N operations to complete before proceeding
 * 
 * Characteristics:
 * - Count down from N to 0
 * - Cannot be reset (one-time use)
 * - Threads wait at await()
 * - Other threads call countDown()
 */

public class CountDownLatchExample {
    
    public static void main(String[] args) throws InterruptedException {
        int workerCount = 5;
        CountDownLatch latch = new CountDownLatch(workerCount);
        
        // Start worker threads
        for (int i = 0; i < workerCount; i++) {
            new Thread(new Worker(latch, i)).start();
        }
        
        // Wait for all workers to finish
        latch.await();
        System.out.println("All workers completed!");
    }
    
    static class Worker implements Runnable {
        private final CountDownLatch latch;
        private final int id;
        
        Worker(CountDownLatch latch, int id) {
            this.latch = latch;
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                System.out.println("Worker " + id + " starting");
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 3000));
                System.out.println("Worker " + id + " finished");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                latch.countDown();  // Signal completion
            }
        }
    }
}
```

### Real-World: Service Startup

```java
public class ServiceManager {
    
    private final List<Service> services;
    
    public void startAllServices() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(services.size());
        
        // Prepare all services
        for (Service service : services) {
            new Thread(() -> {
                try {
                    // Wait for start signal
                    startSignal.await();
                    
                    // Start service
                    service.start();
                    System.out.println(service.getName() + " started");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    doneSignal.countDown();
                }
            }).start();
        }
        
        // Give start signal
        System.out.println("Starting all services...");
        startSignal.countDown();
        
        // Wait for all to complete
        doneSignal.await();
        System.out.println("All services started!");
    }
}
```

### Real-World: Parallel Processing

```java
public class ParallelDataProcessor {
    
    public List<Result> processInParallel(List<Data> dataList) 
            throws InterruptedException {
        
        int threadCount = Runtime.getRuntime().availableProcessors();
        CountDownLatch latch = new CountDownLatch(threadCount);
        
        List<Result> results = new CopyOnWriteArrayList<>();
        int chunkSize = dataList.size() / threadCount;
        
        for (int i = 0; i < threadCount; i++) {
            int start = i * chunkSize;
            int end = (i == threadCount - 1) ? dataList.size() : (i + 1) * chunkSize;
            
            List<Data> chunk = dataList.subList(start, end);
            
            new Thread(() -> {
                try {
                    for (Data data : chunk) {
                        results.add(process(data));
                    }
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        
        latch.await();
        return results;
    }
    
    private Result process(Data data) {
        // Process data
        return new Result();
    }
}
```

### With Timeout

```java
public class TimeoutExample {
    
    public void executeWithTimeout(List<Task> tasks) {
        CountDownLatch latch = new CountDownLatch(tasks.size());
        
        tasks.forEach(task -> new Thread(() -> {
            try {
                task.execute();
            } finally {
                latch.countDown();
            }
        }).start());
        
        try {
            // Wait up to 30 seconds
            boolean completed = latch.await(30, TimeUnit.SECONDS);
            
            if (completed) {
                System.out.println("All tasks completed");
            } else {
                System.out.println("Timeout! Some tasks still running");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## CyclicBarrier

### Basic Usage

```java
import java.util.concurrent.CyclicBarrier;

/**
 * CyclicBarrier - Reusable synchronization point
 * 
 * Use when: Need threads to wait for each other at a common point
 * 
 * Characteristics:
 * - All threads wait at barrier
 * - When last thread arrives, all released
 * - Can be reused (cyclic)
 * - Optional barrier action
 */

public class CyclicBarrierExample {
    
    public static void main(String[] args) {
        int threadCount = 3;
        
        // Barrier action executed when all threads arrive
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
            System.out.println("All threads reached barrier!");
        });
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(new Worker(barrier, i)).start();
        }
    }
    
    static class Worker implements Runnable {
        private final CyclicBarrier barrier;
        private final int id;
        
        Worker(CyclicBarrier barrier, int id) {
            this.barrier = barrier;
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                System.out.println("Worker " + id + " phase 1");
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000));
                
                barrier.await();  // Wait for others
                
                System.out.println("Worker " + id + " phase 2");
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000));
                
                barrier.await();  // Reusable!
                
                System.out.println("Worker " + id + " completed");
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### Real-World: Matrix Multiplication

```java
public class ParallelMatrixMultiplication {
    
    public double[][] multiply(double[][] a, double[][] b) 
            throws InterruptedException, BrokenBarrierException {
        
        int n = a.length;
        int threadCount = Runtime.getRuntime().availableProcessors();
        double[][] result = new double[n][n];
        
        CyclicBarrier barrier = new CyclicBarrier(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            int startRow = i * n / threadCount;
            int endRow = (i + 1) * n / threadCount;
            
            new Thread(() -> {
                try {
                    for (int row = startRow; row < endRow; row++) {
                        for (int col = 0; col < n; col++) {
                            for (int k = 0; k < n; k++) {
                                result[row][col] += a[row][k] * b[k][col];
                            }
                        }
                    }
                    barrier.await();  // Wait for all threads
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
        
        barrier.await();  // Main thread waits too
        return result;
    }
}
```

### Real-World: Simulation with Phases

```java
public class GameSimulation {
    
    private final CyclicBarrier barrier;
    private final List<Player> players;
    
    public GameSimulation(int playerCount) {
        this.players = new ArrayList<>();
        
        // Barrier action: End of round processing
        this.barrier = new CyclicBarrier(playerCount, () -> {
            System.out.println("Round complete! Processing results...");
            processRoundResults();
        });
        
        for (int i = 0; i < playerCount; i++) {
            players.add(new Player(i, barrier));
        }
    }
    
    public void startGame(int rounds) {
        players.forEach(player -> new Thread(player).start());
        
        for (int round = 0; round < rounds; round++) {
            System.out.println("Round " + (round + 1));
            // Players automatically synchronize at barrier
        }
    }
    
    private void processRoundResults() {
        // Calculate scores, update leaderboard, etc.
    }
    
    static class Player implements Runnable {
        private final int id;
        private final CyclicBarrier barrier;
        
        Player(int id, CyclicBarrier barrier) {
            this.id = id;
            this.barrier = barrier;
        }
        
        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // Make move
                    makeMove();
                    
                    // Wait for other players
                    barrier.await();
                }
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        private void makeMove() throws InterruptedException {
            System.out.println("Player " + id + " making move");
            Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));
        }
    }
}
```

---

## Semaphore

### Basic Usage

```java
import java.util.concurrent.Semaphore;

/**
 * Semaphore - Control access to resource pool
 * 
 * Use when: Limit concurrent access to shared resource
 * 
 * Characteristics:
 * - Maintains set of permits
 * - acquire() blocks if no permits available
 * - release() returns permit
 * - Fair vs non-fair acquisition
 */

public class SemaphoreExample {
    
    public static void main(String[] args) {
        // Limit to 3 concurrent threads
        Semaphore semaphore = new Semaphore(3);
        
        for (int i = 0; i < 10; i++) {
            new Thread(new Worker(semaphore, i)).start();
        }
    }
    
    static class Worker implements Runnable {
        private final Semaphore semaphore;
        private final int id;
        
        Worker(Semaphore semaphore, int id) {
            this.semaphore = semaphore;
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                System.out.println("Worker " + id + " waiting for permit");
                semaphore.acquire();
                
                System.out.println("Worker " + id + " acquired permit");
                Thread.sleep(2000);  // Do work
                System.out.println("Worker " + id + " releasing permit");
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                semaphore.release();
            }
        }
    }
}
```

### Real-World: Connection Pool

```java
public class DatabaseConnectionPool {
    
    private final List<Connection> connections;
    private final Semaphore semaphore;
    private final Queue<Connection> availableConnections;
    
    public DatabaseConnectionPool(int poolSize) {
        this.semaphore = new Semaphore(poolSize, true);  // Fair
        this.connections = new ArrayList<>(poolSize);
        this.availableConnections = new ConcurrentLinkedQueue<>();
        
        // Initialize connections
        for (int i = 0; i < poolSize; i++) {
            Connection conn = createConnection();
            connections.add(conn);
            availableConnections.offer(conn);
        }
    }
    
    public Connection acquireConnection() throws InterruptedException {
        semaphore.acquire();
        return availableConnections.poll();
    }
    
    public Connection acquireConnection(long timeout, TimeUnit unit) 
            throws InterruptedException {
        
        if (semaphore.tryAcquire(timeout, unit)) {
            return availableConnections.poll();
        }
        throw new TimeoutException("Could not acquire connection");
    }
    
    public void releaseConnection(Connection connection) {
        if (connection != null) {
            availableConnections.offer(connection);
            semaphore.release();
        }
    }
    
    public int availableConnections() {
        return semaphore.availablePermits();
    }
    
    private Connection createConnection() {
        // Create database connection
        return new Connection();
    }
}

// Usage
DatabaseConnectionPool pool = new DatabaseConnectionPool(10);

Connection conn = pool.acquireConnection();
try {
    // Use connection
    conn.executeQuery("SELECT * FROM users");
} finally {
    pool.releaseConnection(conn);
}
```

### Real-World: Rate Limiter

```java
public class RateLimiter {
    
    private final Semaphore semaphore;
    private final int maxPermits;
    private final long periodMillis;
    
    public RateLimiter(int maxRequests, long periodMillis) {
        this.semaphore = new Semaphore(maxRequests);
        this.maxPermits = maxRequests;
        this.periodMillis = periodMillis;
        
        // Refill permits periodically
        startRefillTask();
    }
    
    public boolean tryAcquire() {
        return semaphore.tryAcquire();
    }
    
    public boolean tryAcquire(long timeout, TimeUnit unit) 
            throws InterruptedException {
        return semaphore.tryAcquire(timeout, unit);
    }
    
    private void startRefillTask() {
        ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(1);
        
        scheduler.scheduleAtFixedRate(() -> {
            int released = maxPermits - semaphore.availablePermits();
            if (released > 0) {
                semaphore.release(released);
            }
        }, periodMillis, periodMillis, TimeUnit.MILLISECONDS);
    }
}

// Usage
RateLimiter limiter = new RateLimiter(100, 1000);  // 100 req/second

if (limiter.tryAcquire()) {
    handleRequest();
} else {
    return "Rate limit exceeded";
}
```

### Acquiring Multiple Permits

```java
public class BulkResourceManager {
    
    private final Semaphore semaphore;
    
    public BulkResourceManager(int totalResources) {
        this.semaphore = new Semaphore(totalResources);
    }
    
    public void processBulk(int resourcesNeeded) 
            throws InterruptedException {
        
        // Acquire multiple permits at once
        semaphore.acquire(resourcesNeeded);
        
        try {
            System.out.println("Processing with " + resourcesNeeded + " resources");
            Thread.sleep(2000);
        } finally {
            semaphore.release(resourcesNeeded);
        }
    }
    
    public boolean tryProcessBulk(int resourcesNeeded, long timeout) {
        try {
            if (semaphore.tryAcquire(resourcesNeeded, timeout, TimeUnit.SECONDS)) {
                try {
                    // Process
                    return true;
                } finally {
                    semaphore.release(resourcesNeeded);
                }
            }
            return false;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

---

## Phaser

### Basic Usage

```java
import java.util.concurrent.Phaser;

/**
 * Phaser - Flexible multi-phase barrier
 * 
 * Use when: Need multiple phases with dynamic party count
 * 
 * Characteristics:
 * - Supports multiple phases
 * - Can dynamically register/deregister parties
 * - More flexible than CyclicBarrier
 * - Can terminate after certain phase
 */

public class PhaserExample {
    
    public static void main(String[] args) {
        Phaser phaser = new Phaser(1);  // Register main thread
        
        for (int i = 0; i < 3; i++) {
            phaser.register();  // Register party
            new Thread(new Worker(phaser, i)).start();
        }
        
        // Wait for phase 0 to complete
        phaser.arriveAndAwaitAdvance();
        System.out.println("Phase 0 completed");
        
        // Wait for phase 1 to complete
        phaser.arriveAndAwaitAdvance();
        System.out.println("Phase 1 completed");
        
        phaser.arriveAndDeregister();  // Main thread done
    }
    
    static class Worker implements Runnable {
        private final Phaser phaser;
        private final int id;
        
        Worker(Phaser phaser, int id) {
            this.phaser = phaser;
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                // Phase 0
                System.out.println("Worker " + id + " phase 0");
                Thread.sleep(1000);
                phaser.arriveAndAwaitAdvance();
                
                // Phase 1
                System.out.println("Worker " + id + " phase 1");
                Thread.sleep(1000);
                phaser.arriveAndAwaitAdvance();
                
                System.out.println("Worker " + id + " completed");
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                phaser.arriveAndDeregister();
            }
        }
    }
}
```

### Real-World: Multi-Stage Pipeline

```java
public class DataPipeline {
    
    public void processPipeline(List<Data> dataList) {
        int processors = Runtime.getRuntime().availableProcessors();
        Phaser phaser = new Phaser(processors) {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("Phase " + phase + " completed");
                return phase >= 2 || registeredParties == 0;  // Terminate after phase 2
            }
        };
        
        for (int i = 0; i < processors; i++) {
            new Thread(new PipelineWorker(phaser, dataList, i)).start();
        }
    }
    
    static class PipelineWorker implements Runnable {
        private final Phaser phaser;
        private final List<Data> dataList;
        private final int id;
        
        PipelineWorker(Phaser phaser, List<Data> dataList, int id) {
            this.phaser = phaser;
            this.dataList = dataList;
            this.id = id;
        }
        
        @Override
        public void run() {
            // Phase 0: Extract
            extract();
            phaser.arriveAndAwaitAdvance();
            
            // Phase 1: Transform
            transform();
            phaser.arriveAndAwaitAdvance();
            
            // Phase 2: Load
            load();
            phaser.arriveAndDeregister();
        }
        
        private void extract() {
            System.out.println("Worker " + id + " extracting");
        }
        
        private void transform() {
            System.out.println("Worker " + id + " transforming");
        }
        
        private void load() {
            System.out.println("Worker " + id + " loading");
        }
    }
}
```

---

## Exchanger

### Basic Usage

```java
import java.util.concurrent.Exchanger;

/**
 * Exchanger - Exchange data between two threads
 * 
 * Use when: Two threads need to exchange data at synchronization point
 * 
 * Characteristics:
 * - Designed for exactly 2 threads
 * - Bidirectional exchange
 * - Blocks until both threads ready
 */

public class ExchangerExample {
    
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();
        
        new Thread(new Producer(exchanger)).start();
        new Thread(new Consumer(exchanger)).start();
    }
    
    static class Producer implements Runnable {
        private final Exchanger<String> exchanger;
        
        Producer(Exchanger<String> exchanger) {
            this.exchanger = exchanger;
        }
        
        @Override
        public void run() {
            try {
                for (int i = 0; i < 3; i++) {
                    String produced = "Data-" + i;
                    System.out.println("Producer: produced " + produced);
                    
                    // Exchange with consumer
                    String consumed = exchanger.exchange(produced);
                    System.out.println("Producer: received " + consumed);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    static class Consumer implements Runnable {
        private final Exchanger<String> exchanger;
        
        Consumer(Exchanger<String> exchanger) {
            this.exchanger = exchanger;
        }
        
        @Override
        public void run() {
            try {
                for (int i = 0; i < 3; i++) {
                    // Exchange with producer
                    String received = exchanger.exchange("ACK-" + i);
                    System.out.println("Consumer: received " + received);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### Real-World: Buffer Swapping

```java
public class BufferSwapper {
    
    private final Exchanger<List<Data>> exchanger = new Exchanger<>();
    
    public void start() {
        new Thread(new Producer(exchanger)).start();
        new Thread(new Consumer(exchanger)).start();
    }
    
    static class Producer implements Runnable {
        private final Exchanger<List<Data>> exchanger;
        private List<Data> currentBuffer;
        
        Producer(Exchanger<List<Data>> exchanger) {
            this.exchanger = exchanger;
            this.currentBuffer = new ArrayList<>();
        }
        
        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // Fill buffer
                    for (int i = 0; i < 10; i++) {
                        currentBuffer.add(produceData());
                    }
                    
                    // Exchange full buffer for empty one
                    currentBuffer = exchanger.exchange(currentBuffer);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        private Data produceData() {
            return new Data();
        }
    }
    
    static class Consumer implements Runnable {
        private final Exchanger<List<Data>> exchanger;
        private List<Data> currentBuffer;
        
        Consumer(Exchanger<List<Data>> exchanger) {
            this.exchanger = exchanger;
            this.currentBuffer = new ArrayList<>();
        }
        
        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // Exchange empty buffer for full one
                    currentBuffer = exchanger.exchange(currentBuffer);
                    
                    // Process data
                    currentBuffer.forEach(this::consume);
                    currentBuffer.clear();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        private void consume(Data data) {
            // Process data
        }
    }
}
```

---

## ReentrantLock

### Basic Usage vs synchronized

```java
import java.util.concurrent.locks.ReentrantLock;

/**
 * ReentrantLock - Explicit lock with advanced features
 * 
 * Use when: Need try-lock, timed-lock, or interruptible locking
 * 
 * Advantages over synchronized:
 * - tryLock() - non-blocking attempt
 * - tryLock(timeout) - timed locking
 * - lockInterruptibly() - interruptible
 * - Fair locking option
 * - Condition variables
 */

public class ReentrantLockExample {
    
    private final ReentrantLock lock = new ReentrantLock();
    private int counter = 0;
    
    // Basic usage
    public void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();  // MUST unlock in finally block
        }
    }
    
    // Try lock
    public boolean tryIncrement() {
        if (lock.tryLock()) {
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Could not acquire lock
    }
    
    // Timed lock
    public boolean tryIncrementWithTimeout(long timeout, TimeUnit unit) 
            throws InterruptedException {
        
        if (lock.tryLock(timeout, unit)) {
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
    
    // Interruptible lock
    public void incrementInterruptibly() throws InterruptedException {
        lock.lockInterruptibly();  // Can be interrupted while waiting
        try {
            counter++;
        } finally {
            lock.unlock();
        }
    }
}
```

### Fair vs Non-Fair Lock

```java
public class FairLockExample {
    
    // Fair lock - FIFO ordering
    private final ReentrantLock fairLock = new ReentrantLock(true);
    
    // Non-fair lock - better performance, may starve threads
    private final ReentrantLock nonFairLock = new ReentrantLock(false);
    
    public void fairProcess() {
        fairLock.lock();  // Threads acquire in order
        try {
            // Critical section
        } finally {
            fairLock.unlock();
        }
    }
    
    // Check lock status
    public void checkLockStatus() {
        System.out.println("Locked: " + fairLock.isLocked());
        System.out.println("Held by current thread: " + fairLock.isHeldByCurrentThread());
        System.out.println("Queue length: " + fairLock.getQueueLength());
        System.out.println("Fair: " + fairLock.isFair());
    }
}
```

### Condition Variables

```java
public class BoundedBuffer<T> {
    
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait until not full
            }
            
            queue.offer(item);
            notEmpty.signal();  // Signal waiting consumers
            
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait until not empty
            }
            
            T item = queue.poll();
            notFull.signal();  // Signal waiting producers
            return item;
            
        } finally {
            lock.unlock();
        }
    }
}
```

---

## ReadWriteLock

### Basic Usage

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * ReadWriteLock - Separate read and write locks
 * 
 * Use when: Many readers, few writers
 * 
 * Characteristics:
 * - Multiple readers can hold read lock simultaneously
 * - Only one writer can hold write lock
 * - No readers when writer holds lock
 * - Better performance for read-heavy workloads
 */

public class ReadWriteCache<K, V> {
    
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    
    // Read operation - multiple threads can read simultaneously
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    // Write operation - exclusive access
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    public void remove(K key) {
        writeLock.lock();
        try {
            cache.remove(key);
        } finally {
            writeLock.unlock();
        }
    }
    
    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
}
```

---

## StampedLock

### Optimistic Read Lock

```java
import java.util.concurrent.locks.StampedLock;

/**
 * StampedLock - Optimistic read lock (Java 8+)
 * 
 * Use when: Read-heavy workloads, want better performance than ReadWriteLock
 * 
 * Three modes:
 * - Optimistic read - no locking, validate later
 * - Pessimistic read - like ReadWriteLock
 * - Write - exclusive
 */

public class Point {
    
    private double x, y;
    private final StampedLock lock = new StampedLock();
    
    // Optimistic read
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // Optimistic read
        
        double currentX = x;
        double currentY = y;
        
        if (!lock.validate(stamp)) {  // Check if valid
            // Fall back to pessimistic read
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
    
    // Write lock
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
    
    // Read lock (pessimistic)
    public double getX() {
        long stamp = lock.readLock();
        try {
            return x;
        } finally {
            lock.unlockRead(stamp);
        }
    }
}
```

---

## Best Practices

### ✅ DO

1. **Always unlock in finally block**
   ```java
   lock.lock();
   try {
       // Critical section
   } finally {
       lock.unlock();  // MUST be in finally
   }
   ```

2. **Use try-with-resources when possible**
3. **Prefer higher-level constructs**
4. **Use CountDownLatch for one-shot events**
5. **Use CyclicBarrier for recurring synchronization**
6. **Use Semaphore for resource pools**
7. **Use ReadWriteLock for read-heavy workloads**
8. **Set timeouts to avoid deadlocks**
9. **Use fair locks when needed**
10. **Monitor lock contention**

### ❌ DON'T

1. **Don't forget to unlock**
2. **Don't use synchronized and explicit locks together**
3. **Don't hold locks longer than necessary**
4. **Don't call alien methods while holding locks**
5. **Don't acquire multiple locks in different orders**

---

## Conclusion

**Key Takeaways:**

1. **CountDownLatch** - One-shot barrier for waiting
2. **CyclicBarrier** - Reusable barrier for phases
3. **Semaphore** - Resource pool management
4. **Phaser** - Flexible multi-phase synchronization
5. **ReentrantLock** - Explicit lock with advanced features
6. **ReadWriteLock** - Separate read/write locks
7. **StampedLock** - Optimistic locking for performance

**Remember**: Choose the right synchronization primitive for your use case. Higher-level constructs are preferred when they fit the problem!
