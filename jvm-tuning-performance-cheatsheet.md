# ⚙️ JVM Tuning & Performance Cheat Sheet

> **Purpose:** Production-grade JVM configuration, GC selection, profiling, and container optimization. Reference before deploying any Java service or investigating performance issues.
> **Stack context:** Java 21+ / Spring Boot 3.x / Containers (Docker/Kubernetes) / ZGC / G1GC

---

## 📋 The Performance Investigation Framework

Before tuning anything, answer:

| Question | Determines |
|----------|-----------|
| What's the **symptom**? (latency, throughput, CPU, memory) | Investigation area |
| Is it **sudden** or **gradual**? | Bug vs leak vs load growth |
| Does it **correlate** with traffic? | Capacity vs code issue |
| What **changed** recently? | Deploy, config, dependency, traffic pattern |
| What do **metrics** show? (GC time, heap, threads) | Root cause area |
| Can it be **reproduced**? | Profiling opportunity |

---

## 🗑️ Pattern 1: Garbage Collector Selection

### GC Decision Tree

```
What's your priority?
│
├── LATENCY (< 10ms pauses)
│   ├── Heap > 512MB → ZGC (Generational)     ✅ RECOMMENDED
│   └── Heap > 512MB → Shenandoah             ✅ Alternative
│
├── THROUGHPUT (max req/sec, pauses OK)
│   ├── Heap > 4GB   → G1GC                   ✅ Good default
│   └── Heap < 4GB   → Parallel GC            ⚠️ Rare use case
│
├── BALANCED (good latency + throughput)
│   └── Any heap     → G1GC                   ✅ Safe default
│
└── TINY CONTAINER (< 256MB heap)
    └── Serial GC                              ✅ Minimal overhead
```

### GC Comparison

| GC | Max Pause | Throughput | Heap Overhead | Best For |
|----|-----------|-----------|---------------|----------|
| **ZGC (Generational)** | < 1ms | High | ~3-5% | Payment processing, real-time APIs |
| **Shenandoah** | < 10ms | High | ~5-10% | Low latency alternative |
| **G1GC** | 50-200ms | Very high | ~5-10% | General purpose, batch processing |
| **Parallel GC** | 100ms-1s | Highest | ~0% | Pure throughput, batch jobs |
| **Serial GC** | Varies | Lowest | ~0% | Tiny heaps, short-lived processes |

### GC Configuration — Production Templates

```bash
# ── ZGC Generational (Java 21+) — Low latency ──
JAVA_OPTS="
  -XX:+UseZGC
  -XX:+ZGenerational
  -XX:SoftMaxHeapSize=768m        # Preferred max (GC tries to stay below)
  -XX:MaxHeapSize=1g              # Hard max (never exceeds)
  -XX:+ZUncommit                  # Return unused memory to OS
  -XX:ZUncommitDelay=300          # After 5 min idle
"

# ── G1GC — Balanced (Safe Default) ──
JAVA_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200        # Target max pause time
  -XX:G1HeapRegionSize=4m         # Region size (auto-tuned if unset)
  -XX:InitiatingHeapOccupancyPercent=45  # Start concurrent GC earlier
  -XX:G1ReservePercent=15         # Reserve for promotion bursts
  -XX:+G1UseAdaptiveIHOP          # Auto-tune IHOP
  -XX:G1MixedGCCountTarget=8      # Spread mixed GC work
"

# ── Serial GC — Tiny containers ──
JAVA_OPTS="
  -XX:+UseSerialGC
  -XX:MaxHeapSize=128m
"
```

### GC Logging — Always Enable in Production

```bash
JAVA_OPTS="
  -Xlog:gc*:file=/var/log/gc.log:time,level,tags:filecount=5,filesize=10m
  -Xlog:gc+phases=debug:file=/var/log/gc.log
  -Xlog:gc+heap=info:file=/var/log/gc.log
"

# Key things to look for in GC logs:
# 1. Pause time > target      → GC can't keep up
# 2. Allocation rate spikes   → Object churn
# 3. Frequent full GCs        → Heap too small or memory leak
# 4. Promotion failures       → Old gen too small
```

---

## 🐳 Pattern 2: Container-Aware JVM

### The Container Memory Problem

```
Container memory limit: 1024MB
│
├── JVM heap:          ~750MB (MaxRAMPercentage=75%)
├── Metaspace:         ~64MB  (class metadata)
├── Thread stacks:     ~50MB  (50 platform threads × 1MB each)
├── Direct buffers:    ~64MB  (NIO, Netty)
├── JIT code cache:    ~48MB  (compiled code)
├── GC overhead:       ~30MB  (varies by GC)
├── Native memory:     ~20MB  (JNI, OS)
└── ─────────────────────────
    Total:             ~1026MB → OOMKilled!

Fix: MaxRAMPercentage=75% leaves 25% for non-heap
     Or explicitly size each region
```

### Container JVM Configuration

```bash
# ── Production container settings ──
JAVA_OPTS="
  # ── Container awareness ──
  -XX:+UseContainerSupport             # Detect container memory/CPU limits
  -XX:MaxRAMPercentage=75.0            # Use 75% of container memory for heap
  -XX:InitialRAMPercentage=50.0        # Start at 50% (faster startup)
  
  # ── GC ──
  -XX:+UseZGC
  -XX:+ZGenerational
  
  # ── Crash handling ──
  -XX:+ExitOnOutOfMemoryError          # Don't limp — die and let K8s restart
  -XX:+HeapDumpOnOutOfMemoryError      # Capture heap dump before dying
  -XX:HeapDumpPath=/tmp/heapdump.hprof
  
  # ── Memory regions ──
  -XX:MaxMetaspaceSize=128m            # Cap metaspace (prevents leak)
  -XX:ReservedCodeCacheSize=64m        # JIT compiled code
  -XX:MaxDirectMemorySize=128m         # NIO direct buffers
  
  # ── Threads ──
  -XX:ThreadStackSize=512k             # Reduce from default 1MB (safe for most apps)
  
  # ── Startup ──
  -XX:+TieredCompilation               # Progressive JIT (faster startup)
  -XX:TieredStopAtLevel=1              # ONLY for short-lived processes (skip C2)
  
  # ── Entropy ──
  -Djava.security.egd=file:/dev/./urandom  # Faster random number generation
  
  # ── JMX disabled (save memory) ──
  -Dspring.jmx.enabled=false
"
```

### Container Sizing Guide

| Service Type | Container Memory | Heap (75%) | CPU Request | CPU Limit |
|---|---|---|---|---|
| **Lightweight API** | 512MB | 384MB | 250m | 1000m |
| **Standard service** | 1GB | 768MB | 500m | 2000m |
| **Heavy processing** | 2GB | 1.5GB | 1000m | 4000m |
| **Kafka Streams** | 2-4GB | 1.5-3GB | 1000m | 4000m |
| **Batch/ETL** | 4-8GB | 3-6GB | 2000m | 8000m |

### Memory Calculation Worksheet

```
Container Limit:     _________ MB

Heap (75%):          _________ MB  (-XX:MaxRAMPercentage=75)
Metaspace:           128        MB  (-XX:MaxMetaspaceSize=128m)
Thread stacks:       _________ MB  (threads × stack_size)
  Platform threads:  _____ × 512KB = _____MB
  Virtual threads:   negligible (few KB each)
Direct memory:       128        MB  (-XX:MaxDirectMemorySize=128m)
Code cache:          64         MB  (-XX:ReservedCodeCacheSize=64m)
GC overhead:         ~3-10%     MB  (depends on GC)
Native/OS:           ~50        MB  (JNI, mapped files)
────────────────────────────────────
Total:               _________ MB  (must be < container limit)
```

---

## 📊 Pattern 3: Key JVM Metrics to Monitor

### Essential Metrics (Micrometer)

```java
// These are auto-registered by Spring Boot Actuator + Micrometer
// Just ensure prometheus endpoint is enabled

// ── Heap memory ──
jvm_memory_used_bytes{area="heap"}          // Current heap usage
jvm_memory_max_bytes{area="heap"}           // Max heap
jvm_memory_committed_bytes{area="heap"}     // OS-committed heap

// ── Non-heap memory ──
jvm_memory_used_bytes{area="nonheap"}       // Metaspace + code cache
jvm_buffer_memory_used_bytes{id="direct"}   // Direct byte buffers

// ── GC ──
jvm_gc_pause_seconds_sum                    // Total GC pause time
jvm_gc_pause_seconds_count                  // Number of GC pauses
jvm_gc_pause_seconds_max                    // Max GC pause
jvm_gc_memory_promoted_bytes_total          // Old gen promotion rate
jvm_gc_memory_allocated_bytes_total         // Allocation rate

// ── Threads ──
jvm_threads_live_threads                    // Current thread count
jvm_threads_peak_threads                    // Peak since start
jvm_threads_states_threads{state="blocked"} // Blocked threads (BAD if > 0)

// ── Classes ──
jvm_classes_loaded_classes                  // Loaded class count
jvm_classes_unloaded_classes_total          // Unloaded (should be low)
```

### Alert Rules

```yaml
# Heap > 85% — approaching OOM
- alert: HighHeapUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
  for: 5m
  severity: warning

# Heap > 95% — imminent OOM
- alert: CriticalHeapUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.95
  for: 1m
  severity: critical

# GC pause > 500ms (for ZGC, should be < 1ms)
- alert: HighGCPause
  expr: jvm_gc_pause_seconds_max > 0.5
  for: 0m
  severity: warning

# GC time > 10% of wall clock
- alert: ExcessiveGCTime
  expr: rate(jvm_gc_pause_seconds_sum[5m]) > 0.1
  for: 5m
  severity: critical

# Blocked threads > 0
- alert: ThreadsBlocked
  expr: jvm_threads_states_threads{state="blocked"} > 0
  for: 1m
  severity: warning

# Thread count growing
- alert: ThreadLeak
  expr: delta(jvm_threads_live_threads[1h]) > 50
  for: 30m
  severity: warning

# Direct buffer approaching limit
- alert: DirectBufferHigh
  expr: jvm_buffer_memory_used_bytes{id="direct"} / 134217728 > 0.8
  for: 5m
  severity: warning
```

---

## 🔍 Pattern 4: Profiling & Diagnostics

### Java Flight Recorder (JFR) — Production Safe

```bash
# ── Start recording at JVM launch ──
JAVA_OPTS="
  -XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr,
      settings=profile,maxsize=256m
"

# ── Attach to running JVM ──
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr settings=profile

# ── Continuous recording (always-on, circular buffer) ──
JAVA_OPTS="
  -XX:StartFlightRecording=disk=true,maxage=1h,maxsize=512m,
      dumponexit=true,filename=/tmp/recording.jfr,settings=default
"
# 'default' settings = low overhead (~1%)
# 'profile' settings = more detail (~2-3%)

# ── Dump on demand ──
jcmd <pid> JFR.dump filename=/tmp/snapshot.jfr

# ── Analyze with JFR CLI ──
jfr summary recording.jfr
jfr print --events jdk.GarbageCollection recording.jfr
jfr print --events jdk.ThreadSleep --stack-depth 10 recording.jfr
```

### Async-profiler (Low Overhead CPU/Allocation Profiling)

```bash
# CPU profiling (flame graph)
./asprof -d 30 -f /tmp/cpu-flamegraph.html -e cpu <pid>

# Allocation profiling (find object churn)
./asprof -d 30 -f /tmp/alloc-flamegraph.html -e alloc <pid>

# Lock contention profiling
./asprof -d 30 -f /tmp/lock-flamegraph.html -e lock <pid>

# Wall-clock profiling (includes I/O wait)
./asprof -d 30 -f /tmp/wall-flamegraph.html -e wall <pid>
```

### Quick Diagnostics Commands

```bash
# ── Thread dump (find deadlocks, blocked threads) ──
jcmd <pid> Thread.print
# Or for virtual threads:
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json

# ── Heap histogram (what's consuming memory?) ──
jcmd <pid> GC.class_histogram | head -30

# ── Heap dump (for deep analysis) ──
jcmd <pid> GC.heap_dump /tmp/heapdump.hprof

# ── VM flags (current settings) ──
jcmd <pid> VM.flags

# ── System properties ──
jcmd <pid> VM.system_properties

# ── Native memory tracking ──
# Must start JVM with: -XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary
```

---

## 🚀 Pattern 5: Startup Optimization

### Startup Time Breakdown

```
Typical Spring Boot startup:
  JVM init:           500ms
  Class loading:      1-3s
  Spring context:     2-5s
  Auto-configuration: 1-2s
  Bean creation:      1-5s (depends on eager init)
  Kafka consumers:    1-2s
  MongoDB connection: 500ms-2s
  ───────────────────────────
  Total:              7-20s typical

For faster startup:
  1. Lazy initialization
  2. Exclude unused auto-configs
  3. AOT compilation (Spring 3.x)
  4. Class Data Sharing (CDS)
```

### Startup Configuration

```yaml
spring:
  main:
    lazy-initialization: false   # Don't lazy-init in prod (slower first request)
                                 # Use TRUE only for dev/test

  autoconfigure:
    exclude:
      # Exclude auto-configs you don't use
      - org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
      - org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration
```

```bash
# ── Class Data Sharing (CDS) — faster startup ──
# Step 1: Create class list (run once during build)
java -XX:DumpLoadedClassList=classes.lst -jar app.jar --spring.main.lazy-initialization=true &
sleep 10 && kill $!

# Step 2: Create shared archive
java -Xshare:dump -XX:SharedClassListFile=classes.lst -XX:SharedArchiveFile=app-cds.jsa -jar app.jar

# Step 3: Use in production
JAVA_OPTS="-Xshare:on -XX:SharedArchiveFile=app-cds.jsa"

# Typical improvement: 20-40% faster startup
```

### Startup Probe Design

```
WRONG: Startup probe checks full readiness
  → App can't start because DB isn't ready → probe fails → K8s restarts → loop

RIGHT: Startup probe checks only JVM/Spring health
  → App starts, readiness probe handles external deps
  → If DB is down, readiness fails (no traffic) but app stays alive

startupProbe:
  httpGet:
    path: /actuator/health/liveness   ← Only JVM health
    port: 8081
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30               ← Generous: 10 + (5×30) = 160s
```

---

## 🔥 Pattern 6: Common Performance Anti-Patterns

### Memory Anti-Patterns

```java
// ❌ String concatenation in loop — creates garbage
String result = "";
for (var item : items) {
    result += item.toString() + ",";    // New String object per iteration
}

// ✅ StringBuilder
var sb = new StringBuilder(items.size() * 20);  // Pre-size
for (var item : items) {
    if (!sb.isEmpty()) sb.append(",");
    sb.append(item);
}

// ❌ Autoboxing in hot path
Map<Integer, Integer> map = new HashMap<>();     // Integer boxing per put/get
for (int i = 0; i < 1_000_000; i++) {
    map.put(i, i * 2);                           // Boxes both int → Integer
}

// ✅ Primitive-friendly collections (Eclipse Collections, or IntIntHashMap)

// ❌ Loading entire collection into memory
List<Payment> all = paymentRepository.findAll();  // Could be millions

// ✅ Stream/cursor processing
try (var cursor = mongoTemplate.stream(query, Payment.class)) {
    cursor.forEachRemaining(this::process);
}

// ❌ Creating heavy objects in hot loops
for (var event : events) {
    var mapper = new ObjectMapper();              // Expensive to create!
    mapper.readValue(event.payload(), Type.class);
}

// ✅ Reuse immutable/thread-safe objects
private static final ObjectMapper MAPPER = new ObjectMapper();  // Create once
```

### Thread & Concurrency Anti-Patterns

```java
// ❌ synchronized blocks with virtual threads (pins carrier thread)
public synchronized void process(Payment p) { ... }

// ✅ ReentrantLock (virtual thread friendly)
private final ReentrantLock lock = new ReentrantLock();
public void process(Payment p) {
    lock.lock();
    try { ... } finally { lock.unlock(); }
}

// ❌ ThreadLocal with virtual threads (one instance per VT = millions)
private static final ThreadLocal<ExpensiveObject> cache = ...

// ✅ ScopedValue or pass as parameter

// ❌ Blocking call in reactive pipeline
return webClient.get()
    .retrieve()
    .bodyToMono(Response.class)
    .map(r -> repository.findById(r.id()).block());  // BLOCKS reactive thread!

// ✅ Chain reactive operations
return webClient.get()
    .retrieve()
    .bodyToMono(Response.class)
    .flatMap(r -> reactiveRepository.findById(r.id()));
```

### I/O & Network Anti-Patterns

```java
// ❌ N+1 queries
var customers = customerRepo.findAll();
for (var customer : customers) {
    var payments = paymentRepo.findByCustomerId(customer.id()); // Query per customer!
}

// ✅ Batch fetch
var customerIds = customers.stream().map(Customer::id).toList();
var payments = paymentRepo.findByCustomerIdIn(customerIds);   // Single query
var grouped = payments.stream().collect(Collectors.groupingBy(Payment::customerId));

// ❌ No connection pool limits
// Default: unlimited connections → eventually exhausts OS file descriptors

// ✅ Always configure pool bounds
// MongoDB: maxPoolSize=50
// HTTP client: maxConnections=100
// JDBC: maximumPoolSize=20

// ❌ No timeout on HTTP calls
restTemplate.getForObject(url, Response.class);  // Waits forever

// ✅ Always set timeouts
RestClient.builder()
    .requestFactory(new JdkClientHttpRequestFactory(
        HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(3))
            .build()))
    .build();
```

---

## 📐 Pattern 7: Performance Testing Checklist

### Before Going to Production

```
Load Testing:
  [ ] Identified peak traffic patterns (RPS, concurrent users)
  [ ] Load tested at 2x expected peak
  [ ] Measured p50, p95, p99 latencies under load
  [ ] Verified throughput meets SLA
  [ ] Tested sustained load (not just spike)

Memory Testing:
  [ ] Ran extended load test (hours, not minutes)
  [ ] Monitored heap growth — no upward trend
  [ ] Verified GC pause times are acceptable
  [ ] Checked for direct buffer leaks
  [ ] Tested with max payload sizes

Connection Testing:
  [ ] Verified connection pool sizing under load
  [ ] Tested connection pool exhaustion recovery
  [ ] Verified timeout behavior on all external calls
  [ ] Tested with slow downstream dependencies

Container Testing:
  [ ] Verified JVM stays within container memory limit
  [ ] Tested with container CPU throttling
  [ ] Verified OOM behavior (ExitOnOutOfMemoryError)
  [ ] Tested graceful shutdown under load
```

---

## 📏 Quick Reference: JVM Flags Cheat Table

### Memory Flags

| Flag | Purpose | Default | Recommendation |
|------|---------|---------|----------------|
| `-XX:MaxRAMPercentage` | Heap as % of container RAM | 25% | **75%** |
| `-XX:InitialRAMPercentage` | Initial heap as % | 1.5% | **50%** |
| `-XX:MaxMetaspaceSize` | Metaspace cap | Unlimited | **128-256m** |
| `-XX:ReservedCodeCacheSize` | JIT code cache | 48MB | **64-128m** |
| `-XX:MaxDirectMemorySize` | NIO direct buffers | = heap | **128-256m** |
| `-XX:ThreadStackSize` | Stack per thread | 1024k | **512k** |

### GC Flags

| Flag | Purpose | When to Use |
|------|---------|-------------|
| `-XX:+UseZGC -XX:+ZGenerational` | Low latency GC | **Default for services** |
| `-XX:+UseG1GC` | Balanced GC | General purpose |
| `-XX:MaxGCPauseMillis=N` | G1 pause target | With G1GC |
| `-XX:+UseSerialGC` | Minimal overhead | Tiny containers |
| `-XX:SoftMaxHeapSize=N` | ZGC preferred max | Save memory |
| `-XX:+ZUncommit` | Return unused memory | Containers |

### Diagnostic Flags (Always On)

| Flag | Purpose |
|------|---------|
| `-XX:+ExitOnOutOfMemoryError` | Die cleanly on OOM (let K8s restart) |
| `-XX:+HeapDumpOnOutOfMemoryError` | Capture heap dump before dying |
| `-XX:+UseContainerSupport` | Detect container limits |
| `-Xlog:gc*:file=gc.log` | GC logging (low overhead) |
| `-XX:NativeMemoryTracking=summary` | Track native memory (slight overhead) |

---

## 💡 Golden Rules of JVM Performance

```
1.  MEASURE before tuning — premature optimization is the root of all evil.
2.  ZGC Generational is the default — unless you have a specific reason for G1.
3.  MaxRAMPercentage=75 in containers — leave 25% for non-heap, OS, and safety.
4.  ALWAYS ExitOnOutOfMemoryError — a JVM in OOM is a zombie, let Kubernetes restart it.
5.  GC LOGGING is free insurance — always enable, parse when needed.
6.  Profile with JFR in production — 1-2% overhead, invaluable during incidents.
7.  CONNECTION POOLS have limits — unbounded pools = unbounded failure.
8.  TIMEOUTS on everything — unbounded waits are the #1 cause of thread exhaustion.
9.  Virtual threads change the game — but watch for pinning (synchronized) and ThreadLocal abuse.
10. Startup time matters in K8s — CDS, exclude unused auto-config, right probe thresholds.
```

---

*Last updated: February 2026 | Stack: Java 21+ / ZGC / G1GC / Containers / Spring Boot 3.x*
