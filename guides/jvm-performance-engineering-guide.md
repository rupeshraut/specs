# JVM Performance Engineering

A comprehensive guide to JVM internals, garbage collection tuning, memory analysis, profiling techniques, and production performance optimization for Java 21+ applications.

---

## Table of Contents

1. [JVM Architecture](#jvm-architecture)
2. [Memory Management](#memory-management)
3. [Garbage Collection](#garbage-collection)
4. [GC Algorithms](#gc-algorithms)
5. [GC Tuning](#gc-tuning)
6. [Memory Leaks](#memory-leaks)
7. [Thread Analysis](#thread-analysis)
8. [CPU Profiling](#cpu-profiling)
9. [JVM Flags](#jvm-flags)
10. [Container Awareness](#container-awareness)
11. [Profiling Tools](#profiling-tools)
12. [Production Optimization](#production-optimization)

---

## JVM Architecture

### JVM Components

```
┌─────────────────────────────────────────────────────────┐
│                    JVM Architecture                      │
├─────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────┐     │
│  │           Runtime Data Areas                   │     │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────┐     │     │
│  │  │   Heap   │  │  Method  │  │  Stack  │     │     │
│  │  │          │  │   Area   │  │  (per   │     │     │
│  │  │ - Young  │  │          │  │ thread) │     │     │
│  │  │ - Old    │  │ - Class  │  │         │     │     │
│  │  │          │  │   Meta   │  │         │     │     │
│  │  └──────────┘  └──────────┘  └─────────┘     │     │
│  └───────────────────────────────────────────────┘     │
│                                                         │
│  ┌───────────────────────────────────────────────┐     │
│  │         Execution Engine                       │     │
│  │  - JIT Compiler (C1, C2)                      │     │
│  │  - Interpreter                                 │     │
│  │  - Garbage Collector                          │     │
│  └───────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

---

## Memory Management

### Heap Structure

```
┌─────────────────────────────────────────────────────────┐
│                      Java Heap                           │
├──────────────────────────┬──────────────────────────────┤
│      Young Generation    │     Old Generation           │
│  ┌────────┬──────────┐   │                             │
│  │  Eden  │ Survivor │   │   (Tenured Space)           │
│  │        │  S0  S1  │   │                             │
│  │  80%   │  10% 10% │   │         67%                 │
│  └────────┴──────────┘   │                             │
└──────────────────────────┴──────────────────────────────┘

Object Lifecycle:
1. Created in Eden
2. Survives Minor GC → Survivor (S0/S1)
3. Survives multiple GCs → Old Generation
4. Eventually collected by Major GC
```

### Memory Configuration

```bash
# Heap sizing
-Xms4g                    # Initial heap size
-Xmx4g                    # Maximum heap size (set equal to Xms)

# Young generation
-XX:NewRatio=2            # Old:Young = 2:1
-XX:SurvivorRatio=8       # Eden:Survivor = 8:1

# Metaspace (replaced PermGen in Java 8+)
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# Direct memory
-XX:MaxDirectMemorySize=1g

# Stack size per thread
-Xss1m
```

### Memory Analysis

```bash
# Heap dump on OutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/heapdump.hprof

# Print GC details
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags

# Monitor memory usage
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary
```

---

## Garbage Collection

### GC Types

```
Minor GC (Young Generation):
- Fast (milliseconds)
- Stop-The-World
- Triggered when Eden full
- Promotes surviving objects

Major GC (Old Generation):
- Slower (100s of milliseconds to seconds)
- Stop-The-World (except G1, ZGC)
- Triggered when Old Generation full
- Collects entire heap

Full GC:
- Slowest
- Compacts entire heap
- Last resort, indicates problem
```

### GC Monitoring

```java
// Programmatic GC monitoring
ManagementFactory.getGarbageCollectorMXBeans().forEach(gc -> {
    System.out.println("GC Name: " + gc.getName());
    System.out.println("Collection Count: " + gc.getCollectionCount());
    System.out.println("Collection Time: " + gc.getCollectionTime() + "ms");
});

// Memory pool monitoring
ManagementFactory.getMemoryPoolMXBeans().forEach(pool -> {
    MemoryUsage usage = pool.getUsage();
    System.out.println("Pool: " + pool.getName());
    System.out.println("Used: " + usage.getUsed() / 1024 / 1024 + "MB");
    System.out.println("Max: " + usage.getMax() / 1024 / 1024 + "MB");
});
```

---

## GC Algorithms

### G1GC (Default in Java 9+)

```bash
# G1GC Configuration
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # Target pause time
-XX:G1HeapRegionSize=16m          # Region size (1-32MB)
-XX:InitiatingHeapOccupancyPercent=45  # Mixed GC threshold
-XX:G1ReservePercent=10           # Reserve for evacuation
-XX:ConcGCThreads=2               # Concurrent GC threads
-XX:ParallelGCThreads=8           # Parallel GC threads

# When to use G1GC:
# ✅ Heap > 4GB
# ✅ Need predictable pause times
# ✅ High throughput with low latency
# ✅ Production default for most apps
```

### ZGC (Java 11+, Production-ready in Java 15+)

```bash
# ZGC Configuration
-XX:+UseZGC
-XX:+ZGenerational              # Enable generational ZGC (Java 21+)
-XX:SoftMaxHeapSize=4g          # Soft limit
-XX:ConcGCThreads=2
-Xlog:gc*:file=/logs/gc-zgc.log

# ZGC Characteristics:
# ✅ Pause times < 1ms (sub-millisecond)
# ✅ Handles heaps up to 16TB
# ✅ Concurrent compaction
# ✅ No generational (before Java 21)
# ✅ Best for ultra-low latency requirements

# When to use ZGC:
# ✅ Large heaps (>100GB)
# ✅ Ultra-low latency required (< 10ms)
# ✅ Willing to trade throughput for latency
```

### Shenandoah GC

```bash
# Shenandoah Configuration
-XX:+UseShenandoahGC
-XX:ShenandoahGCMode=iu         # Incremental Update mode
-XX:ConcGCThreads=2

# Characteristics:
# ✅ Similar to ZGC (low pause times)
# ✅ Better for smaller heaps (< 100GB)
# ✅ Available in OpenJDK
```

### Parallel GC

```bash
# Parallel GC (High throughput)
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
-XX:MaxGCPauseMillis=100

# When to use:
# ✅ Batch processing
# ✅ Throughput > latency
# ✅ Can tolerate longer pauses
```

---

## GC Tuning

### Tuning Process

```
1. Measure baseline
   └─> JVM flags, GC logs, metrics

2. Identify problems
   └─> Long pauses? High frequency? Full GCs?

3. Set goals
   └─> Max pause time? Throughput target?

4. Adjust parameters
   └─> One at a time

5. Measure again
   └─> Compare to baseline

6. Repeat
```

### Common Tuning Scenarios

```bash
# Scenario 1: Too many Minor GCs
# Problem: Eden too small
# Solution: Increase young generation
-XX:NewRatio=1              # 50% of heap to young gen
-Xmn2g                      # Or set absolute size

# Scenario 2: Long Minor GC pauses
# Problem: Large Eden
# Solution: Decrease young generation or increase survivor space
-XX:NewRatio=3              # 25% of heap to young gen
-XX:SurvivorRatio=6         # Larger survivor space

# Scenario 3: Premature promotion to Old Gen
# Problem: Objects promoted too early
# Solution: Increase survivor space or tenure threshold
-XX:SurvivorRatio=6
-XX:MaxTenuringThreshold=15

# Scenario 4: Frequent Full GCs
# Problem: Old generation filling up
# Solution: Increase heap or investigate memory leaks
-Xmx8g
# Or fix memory leak

# Scenario 5: Long GC pauses
# Problem: Wrong GC algorithm
# Solution: Switch to G1GC or ZGC
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

### G1GC Tuning

```bash
# Start with defaults
-XX:+UseG1GC
-Xms4g -Xmx4g
-XX:MaxGCPauseMillis=200

# If pauses still too long:
-XX:MaxGCPauseMillis=100    # More aggressive target

# If throughput too low:
-XX:MaxGCPauseMillis=500    # Relax pause target
-XX:ConcGCThreads=4         # More concurrent threads

# If mixed GC cycles too frequent:
-XX:InitiatingHeapOccupancyPercent=50  # Start mixed GC later

# Monitor with:
-Xlog:gc*,gc+heap=trace:file=/logs/gc.log:time,uptime,level,tags
```

---

## Memory Leaks

### Detecting Memory Leaks

```java
// Signs of memory leak:
// 1. Increasing Old Gen usage over time
// 2. Frequent Full GCs
// 3. Eventually OutOfMemoryError

// Heap dump analysis
jmap -dump:live,format=b,file=heap.hprof <pid>

// Analyze with Eclipse MAT or VisualVM
```

### Common Memory Leak Patterns

```java
// ❌ BAD: Static collections never cleared
public class CacheManager {
    private static final Map<String, Object> cache = new HashMap<>();

    public void cache(String key, Object value) {
        cache.put(key, value);  // Never removed!
    }
}

// ✅ GOOD: Use weak references or size-limited cache
public class CacheManager {
    private final Map<String, Object> cache = new ConcurrentHashMap<>();
    private final int maxSize = 10000;

    public void cache(String key, Object value) {
        if (cache.size() >= maxSize) {
            // Remove oldest entries
            cache.keySet().stream()
                .limit(1000)
                .forEach(cache::remove);
        }
        cache.put(key, value);
    }
}

// Or use Caffeine/Guava cache with size limits
LoadingCache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> loadValue(key));
```

```java
// ❌ BAD: ThreadLocal not cleaned up
public class UserContext {
    private static final ThreadLocal<User> userContext = new ThreadLocal<>();

    public static void setUser(User user) {
        userContext.set(user);
        // Never removed!
    }
}

// ✅ GOOD: Clean up ThreadLocal
public class UserContext {
    private static final ThreadLocal<User> userContext = new ThreadLocal<>();

    public static void setUser(User user) {
        userContext.set(user);
    }

    public static void clear() {
        userContext.remove();  // Always clean up
    }
}

// Use in filter
@Override
public void doFilter(ServletRequest request, ServletResponse response,
                     FilterChain chain) throws IOException, ServletException {
    try {
        UserContext.setUser(extractUser(request));
        chain.doFilter(request, response);
    } finally {
        UserContext.clear();  // Critical!
    }
}
```

---

## Thread Analysis

### Thread Dumps

```bash
# Generate thread dump
jstack <pid> > thread-dump.txt

# Analyze for:
# - Deadlocks
# - Thread states (BLOCKED, WAITING)
# - CPU-intensive threads
```

### Thread States

```java
// Thread state analysis
ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();

long[] threadIds = threadBean.getAllThreadIds();
for (long threadId : threadIds) {
    ThreadInfo info = threadBean.getThreadInfo(threadId);

    if (info.getThreadState() == Thread.State.BLOCKED) {
        System.out.println("Blocked thread: " + info.getThreadName());
        System.out.println("Blocked on: " + info.getLockName());
        System.out.println("Lock owner: " + info.getLockOwnerName());
    }
}

// CPU usage per thread
for (long threadId : threadIds) {
    long cpuTime = threadBean.getThreadCpuTime(threadId);
    ThreadInfo info = threadBean.getThreadInfo(threadId);
    System.out.println(info.getThreadName() + ": " +
        cpuTime / 1_000_000 + "ms");
}
```

---

## CPU Profiling

### Finding CPU Hotspots

```bash
# 1. Find high CPU thread
top -H -p <pid>

# 2. Convert thread ID to hex
printf "%x\n" <thread-id>

# 3. Find in thread dump
jstack <pid> | grep <hex-thread-id>

# 4. Async Profiler (best tool)
./profiler.sh -d 60 -f flamegraph.html <pid>
```

### Java Flight Recorder (JFR)

```bash
# Start recording
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Dump recording
jcmd <pid> JFR.dump filename=recording.jfr

# Stop recording
jcmd <pid> JFR.stop

# Analyze with JDK Mission Control
jmc recording.jfr
```

---

## JVM Flags

### Production Flags

```bash
#!/bin/bash
# Production JVM configuration

JAVA_OPTS=""

# Memory
JAVA_OPTS="$JAVA_OPTS -Xms4g -Xmx4g"
JAVA_OPTS="$JAVA_OPTS -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"

# GC
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:InitiatingHeapOccupancyPercent=45"

# GC Logging
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:file=/logs/gc-%t.log:time,uptime,level,tags"
JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags"

# Crash dumps
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:HeapDumpPath=/dumps/heapdump-%p.hprof"
JAVA_OPTS="$JAVA_OPTS -XX:ErrorFile=/logs/hs_err_pid%p.log"

# Performance
JAVA_OPTS="$JAVA_OPTS -XX:+UseStringDeduplication"
JAVA_OPTS="$JAVA_OPTS -XX:+OptimizeStringConcat"

# JFR (always on in production)
JAVA_OPTS="$JAVA_OPTS -XX:StartFlightRecording=disk=true,maxsize=1g,maxage=24h"

# Container awareness (if in Docker/K8s)
JAVA_OPTS="$JAVA_OPTS -XX:+UseContainerSupport"
JAVA_OPTS="$JAVA_OPTS -XX:MaxRAMPercentage=75.0"

java $JAVA_OPTS -jar application.jar
```

---

## Container Awareness

### Docker/Kubernetes JVM Settings

```bash
# Let JVM detect container limits
-XX:+UseContainerSupport        # Default in Java 11+
-XX:MaxRAMPercentage=75.0       # Use 75% of container memory
-XX:InitialRAMPercentage=50.0   # Initial heap
-XX:MinRAMPercentage=25.0       # Minimum

# Example: 2GB container
# Max heap = 2GB * 0.75 = 1.5GB
# Remaining 500MB for metaspace, threads, native memory
```

### Kubernetes Resource Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  containers:
    - name: app
      image: payment-service:1.0
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "2Gi"      # Hard limit
          cpu: "2000m"

      env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
```

---

## Production Optimization

### Optimization Checklist

```
✅ Memory
- [ ] Heap sized appropriately (not too large!)
- [ ] Xms = Xmx (prevent resizing)
- [ ] Container-aware settings
- [ ] Metaspace limits set

✅ GC
- [ ] Right GC algorithm chosen
- [ ] GC logging enabled
- [ ] Pause time goals set
- [ ] No frequent Full GCs

✅ Monitoring
- [ ] JFR always-on recording
- [ ] GC logs rotated
- [ ] Heap dumps on OOM
- [ ] Metrics exported (Micrometer)

✅ Performance
- [ ] String deduplication enabled
- [ ] Class data sharing considered
- [ ] Native memory tracked
- [ ] Thread dumps accessible
```

### Performance Metrics

```java
@Component
public class JvmMetrics {

    private final MeterRegistry registry;

    @PostConstruct
    public void init() {
        // Memory metrics
        Gauge.builder("jvm.memory.heap.used", this::getHeapUsed)
            .register(registry);

        Gauge.builder("jvm.memory.heap.max", this::getHeapMax)
            .register(registry);

        // GC metrics
        Gauge.builder("jvm.gc.pause", this::getGcPauseTime)
            .register(registry);

        // Thread metrics
        Gauge.builder("jvm.threads.live", this::getLiveThreads)
            .register(registry);
    }

    private long getHeapUsed() {
        return ManagementFactory.getMemoryMXBean()
            .getHeapMemoryUsage()
            .getUsed();
    }

    private long getGcPauseTime() {
        return ManagementFactory.getGarbageCollectorMXBeans().stream()
            .mapToLong(GarbageCollectorMXBean::getCollectionTime)
            .sum();
    }
}
```

---

## Best Practices

### ✅ DO

- Set Xms = Xmx in production
- Enable GC logging
- Use container-aware JVM settings
- Monitor GC pause times
- Generate heap dumps on OOM
- Use G1GC for most applications
- Profile before optimizing
- Keep JVM updated
- Monitor memory trends
- Use JFR for production profiling

### ❌ DON'T

- Use very large heaps (> 32GB) without ZGC
- Disable GC logging
- Ignore Full GCs
- Tune blindly without measuring
- Set Xmx without container limits
- Use Serial GC in production
- Skip memory leak investigation
- Optimize prematurely
- Ignore thread dumps
- Forget to clean up ThreadLocals

---

*This guide provides comprehensive JVM performance engineering patterns. Always measure before and after changes, and optimize based on actual production metrics.*
