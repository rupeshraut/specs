# Effective Java Application Profiling and Performance Analysis

A comprehensive guide to profiling Java applications using JFR, Async Profiler, and GC log analysis.

---

## Table of Contents

1. [Profiling Fundamentals](#profiling-fundamentals)
2. [Java Flight Recorder (JFR)](#java-flight-recorder-jfr)
3. [JFR Advanced Features](#jfr-advanced-features)
4. [Async Profiler](#async-profiler)
5. [GC Log Analysis](#gc-log-analysis)
6. [Memory Profiling](#memory-profiling)
7. [CPU Profiling](#cpu-profiling)
8. [I/O and Lock Profiling](#io-and-lock-profiling)
9. [Production Profiling](#production-profiling)
10. [Performance Optimization Workflow](#performance-optimization-workflow)
11. [Tools Comparison](#tools-comparison)
12. [Best Practices](#best-practices)

---

## Profiling Fundamentals

### Why Profile?

```
Performance Issues to Identify:
┌─────────────────────────────────────┐
│ High CPU Usage                      │
│ - Hot methods                       │
│ - Inefficient algorithms            │
│ - Thread contention                 │
├─────────────────────────────────────┤
│ Memory Problems                     │
│ - Memory leaks                      │
│ - Excessive allocations             │
│ - Large object retention            │
├─────────────────────────────────────┤
│ Garbage Collection Issues           │
│ - Long GC pauses                    │
│ - Frequent GC cycles                │
│ - Old gen growth                    │
├─────────────────────────────────────┤
│ I/O Bottlenecks                     │
│ - Slow database queries             │
│ - Network latency                   │
│ - File system operations            │
├─────────────────────────────────────┤
│ Lock Contention                     │
│ - Thread blocking                   │
│ - Deadlocks                         │
│ - Synchronized methods              │
└─────────────────────────────────────┘
```

### Profiling Types

```
┌──────────────────┬─────────────────────────────────────────────────┐
│ Profiling Type   │ Use Case                                        │
├──────────────────┼─────────────────────────────────────────────────┤
│ Sampling         │ Low overhead, production-safe                   │
│                  │ Finds hot methods, approximate results          │
├──────────────────┼─────────────────────────────────────────────────┤
│ Instrumentation  │ Precise measurements, high overhead             │
│                  │ Method entry/exit counts, exact timing          │
├──────────────────┼─────────────────────────────────────────────────┤
│ Event-based      │ JFR, specific events (GC, allocation, etc.)     │
│                  │ Comprehensive, configurable overhead            │
├──────────────────┼─────────────────────────────────────────────────┤
│ Heap Analysis    │ Memory dumps, retention analysis                │
│                  │ Find memory leaks, large objects                │
└──────────────────┴─────────────────────────────────────────────────┘
```

---

## Java Flight Recorder (JFR)

### 1. **JFR Basics**

```bash
# Enable JFR (JDK 11+, enabled by default)
# No special flags needed for JDK 11+

# Start recording on JVM startup
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp

# With settings
java -XX:StartFlightRecording=settings=profile,duration=120s,filename=recording.jfr MyApp

# Multiple options
java \
  -XX:StartFlightRecording=\
duration=300s,\
filename=/path/to/recording.jfr,\
settings=profile,\
dumponexit=true,\
maxsize=500M \
  MyApp
```

### 2. **JFR Settings Profiles**

```bash
# Location of settings files
# $JAVA_HOME/lib/jfr/

# Available profiles:
# - default.jfc: Low overhead (<1%), basic events
# - profile.jfc: Medium overhead (~2%), more events

# View available profiles
jcmd <PID> JFR.configure

# Start with custom settings
java -XX:StartFlightRecording=settings=/path/to/custom.jfc MyApp
```

### 3. **Controlling JFR at Runtime**

```bash
# Find JVM process
jps -l

# Start recording
jcmd <PID> JFR.start name=MyRecording settings=profile duration=60s filename=recording.jfr

# Check status
jcmd <PID> JFR.check

# Dump current recording
jcmd <PID> JFR.dump name=MyRecording filename=dump.jfr

# Stop recording
jcmd <PID> JFR.stop name=MyRecording

# Example: Start profiling a running application
jps -l
# Output: 12345 com.example.MyApplication

jcmd 12345 JFR.start name=production-profile settings=profile duration=300s filename=/tmp/prod-profile.jfr

# Wait for recording to complete or dump it
jcmd 12345 JFR.dump name=production-profile filename=/tmp/prod-dump.jfr

# Stop the recording
jcmd 12345 JFR.stop name=production-profile
```

### 4. **Analyzing JFR Files**

```bash
# Using JDK Mission Control (JMC)
jmc recording.jfr

# Command-line analysis with jfr tool (JDK 14+)
jfr print recording.jfr

# Print specific events
jfr print --events CPULoad recording.jfr

# Print as JSON
jfr print --json recording.jfr > recording.json

# Print statistics
jfr summary recording.jfr

# Extract metadata
jfr metadata recording.jfr
```

### 5. **JFR Event Categories**

```
Key Event Types:

CPU Events:
├─ Method Profiling Sample
├─ CPU Load
├─ Thread CPU Load
└─ CPU Information

Memory Events:
├─ Allocation
├─ TLAB Allocation
├─ Object Allocation Sample
├─ GC Events (pause, phase, configuration)
└─ Heap Summary

Thread Events:
├─ Thread Start/End
├─ Thread Park
├─ Thread Sleep
├─ Monitor Enter/Wait
└─ Java Monitor Blocked

I/O Events:
├─ File Read/Write
├─ Socket Read/Write
├─ Class Loading
└─ Compilation

Application Events:
├─ Exception Statistics
├─ Throwing Exceptions
└─ Error Statistics
```

### 6. **Programmatic JFR Control**

```java
import jdk.jfr.*;

public class JFRExample {
    
    // Enable JFR programmatically
    public void startRecording() throws Exception {
        Configuration config = Configuration.getConfiguration("profile");
        
        Recording recording = new Recording(config);
        recording.setName("MyRecording");
        recording.setMaxAge(Duration.ofMinutes(10));
        recording.setMaxSize(100_000_000); // 100 MB
        recording.setDestination(Path.of("/tmp/recording.jfr"));
        recording.setDumpOnExit(true);
        
        recording.start();
        
        // Do work...
        
        recording.stop();
        recording.close();
    }
    
    // Custom JFR event
    @Name("com.example.DatabaseQuery")
    @Label("Database Query")
    @Category("Application")
    @StackTrace(false)
    public static class DatabaseQueryEvent extends Event {
        
        @Label("Query")
        public String query;
        
        @Label("Duration")
        public long duration;
        
        @Label("Rows Returned")
        public int rowCount;
    }
    
    // Using custom event
    public void executeQuery(String sql) {
        DatabaseQueryEvent event = new DatabaseQueryEvent();
        event.begin();
        
        try {
            // Execute query
            ResultSet rs = statement.executeQuery(sql);
            int count = 0;
            while (rs.next()) {
                count++;
            }
            
            event.query = sql;
            event.rowCount = count;
            
        } finally {
            event.end();
            if (event.shouldCommit()) {
                event.commit();
            }
        }
    }
    
    // Streaming API (JDK 14+)
    public void streamingAnalysis() throws Exception {
        try (RecordingStream rs = new RecordingStream()) {
            
            // Subscribe to specific events
            rs.onEvent("jdk.CPULoad", event -> {
                double machineLoad = event.getDouble("machineTotal");
                double jvmLoad = event.getDouble("jvmUser") + event.getDouble("jvmSystem");
                
                if (jvmLoad > 0.8) {
                    System.out.println("High CPU load: " + jvmLoad);
                }
            });
            
            rs.onEvent("jdk.GarbageCollection", event -> {
                long duration = event.getDuration().toMillis();
                if (duration > 100) {
                    System.out.println("Long GC pause: " + duration + "ms");
                }
            });
            
            rs.onEvent("jdk.ObjectAllocationSample", event -> {
                String className = event.getClass("objectClass").getName();
                long weight = event.getLong("weight");
                
                System.out.println("Allocation: " + className + ", size: " + weight);
            });
            
            rs.start();
        }
    }
}
```

---

## JFR Advanced Features

### 1. **Custom JFR Settings File**

```xml
<!-- custom-profile.jfc -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration version="2.0" label="Custom Profile" 
              description="Custom profiling configuration"
              provider="MyCompany">
  
  <!-- Method Profiling -->
  <event name="jdk.ExecutionSample">
    <setting name="enabled">true</setting>
    <setting name="period">10 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <!-- Object Allocation -->
  <event name="jdk.ObjectAllocationSample">
    <setting name="enabled">true</setting>
    <setting name="throttle">150/s</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <!-- GC Events -->
  <event name="jdk.GarbageCollection">
    <setting name="enabled">true</setting>
    <setting name="threshold">20 ms</setting>
  </event>
  
  <!-- Exception Events -->
  <event name="jdk.JavaExceptionThrow">
    <setting name="enabled">true</setting>
    <setting name="threshold">0 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <!-- File I/O -->
  <event name="jdk.FileRead">
    <setting name="enabled">true</setting>
    <setting name="threshold">10 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <event name="jdk.FileWrite">
    <setting name="enabled">true</setting>
    <setting name="threshold">10 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <!-- Socket I/O -->
  <event name="jdk.SocketRead">
    <setting name="enabled">true</setting>
    <setting name="threshold">10 ms</setting>
  </event>
  
  <event name="jdk.SocketWrite">
    <setting name="enabled">true</setting>
    <setting name="threshold">10 ms</setting>
  </event>
  
</configuration>
```

### 2. **JFR Analysis with jfr-cli**

```bash
# Install jfr-cli (https://github.com/moditect/jfr-cli)

# Print event summary
jfr print --events jdk.ExecutionSample recording.jfr

# Filter by time range
jfr print --events jdk.GarbageCollection \
  --start-time "2024-01-15T10:00:00" \
  --end-time "2024-01-15T10:05:00" \
  recording.jfr

# Create flame graph from JFR
jfr print --events jdk.ExecutionSample recording.jfr | \
  grep -v "^#" | \
  awk '{print $NF}' | \
  flamegraph.pl > flamegraph.svg

# Extract allocation hot spots
jfr print --events jdk.ObjectAllocationSample recording.jfr | \
  awk '{print $NF}' | \
  sort | uniq -c | sort -rn | head -20
```

---

## Async Profiler

### 1. **Installation and Basic Usage**

```bash
# Download async-profiler
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar -xzf async-profiler-3.0-linux-x64.tar.gz

# Profile CPU
./profiler.sh -d 30 -f cpu-profile.html <PID>

# Profile allocations
./profiler.sh -d 30 -e alloc -f alloc-profile.html <PID>

# Profile locks
./profiler.sh -d 30 -e lock -f lock-profile.html <PID>

# Wall-clock profiling (includes I/O waits)
./profiler.sh -d 30 -e wall -f wall-profile.html <PID>

# Generate flame graph
./profiler.sh -d 30 -f flamegraph.svg <PID>

# Differential flame graph (before/after optimization)
./profiler.sh -d 30 -f before.jfr <PID>
# Make changes
./profiler.sh -d 30 -f after.jfr <PID>

# Compare
./profiler.sh diff before.jfr after.jfr > diff-flamegraph.html
```

### 2. **Advanced Async Profiler Options**

```bash
# Sample at different interval (default: 10ms)
./profiler.sh -d 30 -i 1ms -f profile.html <PID>

# Include specific threads
./profiler.sh -d 30 -t -f profile.html <PID>

# Filter by thread name
./profiler.sh -d 30 --filter 'worker-*' -f profile.html <PID>

# Profile specific methods
./profiler.sh -d 30 --include 'com.example.*' -f profile.html <PID>
./profiler.sh -d 30 --exclude 'java.*' -f profile.html <PID>

# Allocation profiling with live objects only
./profiler.sh -d 30 -e alloc --live -f alloc-live.html <PID>

# Profile until specific allocation threshold
./profiler.sh -e alloc --total -f alloc.html <PID>

# Convert to different formats
./profiler.sh -d 30 -o jfr -f output.jfr <PID>
./profiler.sh -d 30 -o collapsed -f output.txt <PID>

# Profile with JFR compatibility
./profiler.sh -d 30 -o jfr -f recording.jfr <PID>
# Can then open in JMC
```

### 3. **Async Profiler Programmatic API**

```java
import one.profiler.AsyncProfiler;

public class ProfilingExample {
    
    private static final AsyncProfiler profiler = AsyncProfiler.getInstance();
    
    public void profileCPU() throws Exception {
        // Start CPU profiling
        profiler.start("cpu", 1_000_000); // 1ms interval
        
        // Code to profile
        doWork();
        
        // Stop and dump
        profiler.stop();
        profiler.dumpFlat("/tmp/cpu-profile.txt");
        profiler.dumpTraces("/tmp/cpu-traces.txt", 100);
    }
    
    public void profileAllocations() throws Exception {
        // Profile allocations
        profiler.start("alloc", 524288); // Sample every 512KB
        
        doWork();
        
        profiler.stop();
        profiler.dumpFlat("/tmp/alloc-profile.txt");
    }
    
    public void profileWithContext() throws Exception {
        // Start profiling
        profiler.execute("start,event=cpu,interval=1ms,file=/tmp/profile.jfr");
        
        doWork();
        
        // Stop profiling
        profiler.execute("stop");
    }
}
```

### 4. **Reading Async Profiler Output**

```
# Flame Graph Interpretation

Width: Time spent (or allocations made)
├─ Wider = More time/allocations
└─ Narrow = Less time/allocations

Color: Usually not significant (just for visibility)
├─ Can be configured to show different aspects
└─ Red/warm colors often = Java code
    Blue/cool colors often = Native/kernel code

Stack Depth: Call hierarchy
├─ Bottom = Root callers
├─ Top = Leaf methods (actual work)
└─ Follow path to see call chain

Reading Example:
┌─────────────────────────────────────────────────┐
│ main                                            │ ← Entry point
├─────────────────────────────────────────────────┤
│ processOrders (80%)                             │ ← Most time here
├─────────────────┬───────────────────────────────┤
│ calculatePrice  │ sendNotification (20%)        │
│ (60%)           │                               │
└─────────────────┴───────────────────────────────┘

Focus: calculatePrice taking 60% of processOrders time
```

---

## GC Log Analysis

### 1. **Enabling GC Logging**

```bash
# JDK 8 (old flags, deprecated)
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps \
-XX:+PrintGCTimeStamps \
-Xloggc:/path/to/gc.log \
-XX:+UseGCLogFileRotation \
-XX:NumberOfGCLogFiles=5 \
-XX:GCLogFileSize=20M

# JDK 9+ (Unified JVM Logging)
-Xlog:gc*:file=/path/to/gc.log:time,level,tags:filecount=5,filesize=20M

# Detailed GC logging (JDK 9+)
-Xlog:gc*,gc+age=trace,safepoint:file=/path/to/gc.log:time,level,tags:filecount=10,filesize=100M

# Specific GC events
-Xlog:gc+heap=debug:file=/path/to/gc-heap.log
-Xlog:gc+metaspace=trace:file=/path/to/gc-metaspace.log

# With G1GC specifics
-Xlog:gc*,gc+ergo*=trace,gc+age*=trace:file=/path/to/g1-gc.log:time,level,tags
```

### 2. **GC Log Format (JDK 11+ with G1GC)**

```
Example G1GC log entry:

[2024-01-15T10:30:45.123+0000][info][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause) 512M->128M(1024M) 12.345ms

Breaking it down:
├─ [2024-01-15T10:30:45.123+0000] - Timestamp
├─ [info] - Log level
├─ [gc] - Tag
├─ GC(42) - GC event number
├─ Pause Young - GC type
├─ (Normal) - Reason
├─ (G1 Evacuation Pause) - Specific phase
├─ 512M->128M(1024M) - Heap: before->after(total)
└─ 12.345ms - Pause time

Full GC example:
[2024-01-15T10:30:45.500+0000][info][gc] GC(43) Pause Full (Allocation Failure) 1000M->800M(1024M) 1234.567ms

Old Gen growth:
[2024-01-15T10:30:46.000+0000][info][gc,heap] GC(44) Old regions: 100->120
```

### 3. **Analyzing GC Logs with gceasy.io**

```bash
# Upload gc.log to https://gceasy.io for visual analysis

# Or use GCViewer (offline tool)
java -jar gcviewer.jar /path/to/gc.log

# Key Metrics to Analyze:

1. GC Frequency
   ├─ How often GC occurs
   ├─ Young GC frequency
   └─ Full GC frequency (should be rare!)

2. GC Pause Times
   ├─ Average pause time
   ├─ Maximum pause time
   ├─ 95th/99th percentile
   └─ Pause time distribution

3. Heap Usage
   ├─ Young gen usage over time
   ├─ Old gen usage trend
   ├─ Heap size vs used
   └─ Allocation rate

4. Throughput
   ├─ Application time vs GC time
   ├─ GC overhead percentage
   └─ Should be > 95% application time

5. Memory Leaks
   ├─ Old gen growing continuously
   ├─ Full GCs not reclaiming memory
   └─ Heap usage trend upward
```

### 4. **GC Log Analysis Script**

```bash
#!/bin/bash
# analyze-gc.sh

GC_LOG=$1

echo "=== GC Log Analysis ==="
echo "Log file: $GC_LOG"
echo ""

# Count GC events
echo "GC Event Summary:"
echo "Young GC count: $(grep -c "Pause Young" $GC_LOG)"
echo "Full GC count: $(grep -c "Pause Full" $GC_LOG)"
echo "Total GC events: $(grep -c "Pause" $GC_LOG)"
echo ""

# Calculate total pause time
echo "Pause Time Analysis:"
TOTAL_PAUSE=$(grep "Pause" $GC_LOG | \
  awk '{print $NF}' | \
  sed 's/ms//' | \
  awk '{sum+=$1} END {print sum}')
echo "Total pause time: ${TOTAL_PAUSE}ms"

# Find longest pause
LONGEST_PAUSE=$(grep "Pause" $GC_LOG | \
  awk '{print $NF}' | \
  sed 's/ms//' | \
  sort -rn | \
  head -1)
echo "Longest pause: ${LONGEST_PAUSE}ms"

# Heap usage trend
echo ""
echo "Heap Usage (last 10 GCs):"
grep "Pause" $GC_LOG | \
  tail -10 | \
  awk '{print $NF, $(NF-1)}'

# Check for memory leak indicators
echo ""
echo "Memory Leak Indicators:"
FULL_GC_COUNT=$(grep -c "Pause Full" $GC_LOG)
if [ $FULL_GC_COUNT -gt 10 ]; then
  echo "WARNING: $FULL_GC_COUNT Full GCs detected!"
  echo "Check for memory leaks"
fi

# Old gen growth
echo ""
echo "Old Gen Growth:"
grep "Old regions:" $GC_LOG | \
  awk '{print $NF}' | \
  sed 's/->.*//' | \
  tail -20
```

### 5. **Interpreting GC Patterns**

```
Common GC Issues:

1. Frequent Young GCs
   Symptom: Young GC every few seconds
   Cause: High allocation rate
   Solution: Increase Young gen size (-Xmn)
             Optimize allocation-heavy code

2. Long Young GC Pauses
   Symptom: Young GC > 100ms
   Cause: Large Young gen, many live objects
   Solution: Reduce Young gen size
             Review object retention

3. Frequent Full GCs
   Symptom: Full GC every few minutes
   Cause: Old gen filling up
   Solution: Increase heap size (-Xmx)
             Fix memory leaks
             Tune promotion threshold

4. Old Gen Growing
   Pattern: Old gen usage steadily increasing
   Cause: Memory leak!
   Solution: Take heap dump
             Analyze with MAT/JProfiler
             Find and fix leak

5. Allocation Failure
   Message: "Allocation Failure" in log
   Cause: Cannot allocate in Young gen
   Solution: Increase Young gen size
             Reduce allocation rate

6. Humongous Allocations (G1GC)
   Message: "Humongous Allocation"
   Cause: Objects > 50% of G1 region size
   Solution: Increase G1 region size
             Avoid large object allocations
             -XX:G1HeapRegionSize=4M
```

---

## Memory Profiling

### 1. **Heap Dumps**

```bash
# Trigger heap dump manually
jcmd <PID> GC.heap_dump /path/to/heap.hprof

# Or with jmap
jmap -dump:live,format=b,file=/path/to/heap.hprof <PID>

# Automatic heap dump on OutOfMemoryError
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/path/to/dumps \
     MyApp

# Analyze with Eclipse MAT
mat heap.hprof

# Or with jhat (deprecated, but simple)
jhat heap.hprof
# Then visit http://localhost:7000

# OQL queries in MAT
# Find all instances of a class
select * from com.example.User

# Find objects by size
select * from instanceof java.lang.String s where s.@retainedHeapSize > 1000000

# Find objects by field
select * from com.example.User u where u.email.toString() like ".*@gmail.com"
```

### 2. **Native Memory Tracking (NMT)**

```bash
# Enable NMT
java -XX:NativeMemoryTracking=summary MyApp

# Get NMT summary
jcmd <PID> VM.native_memory summary

# Get detailed NMT report
jcmd <PID> VM.native_memory detail

# Baseline for comparison
jcmd <PID> VM.native_memory baseline

# Compare against baseline
jcmd <PID> VM.native_memory summary.diff

# Example NMT output interpretation:
# Native Memory Tracking:
# 
# Total: reserved=5GB, committed=3GB
# -                 Java Heap (reserved=4GB, committed=2GB)
# -                     Class (reserved=512MB, committed=256MB)
# -                    Thread (reserved=256MB, committed=128MB)
# -                      Code (reserved=128MB, committed=64MB)
# -                        GC (reserved=64MB, committed=32MB)
# -                  Internal (reserved=32MB, committed=16MB)
```

### 3. **Finding Memory Leaks with JFR**

```java
// Enable allocation profiling
java -XX:StartFlightRecording=settings=profile,duration=300s,filename=alloc.jfr MyApp

// Analyze in JMC:
// 1. Open JFR file in JMC
// 2. Go to "Memory" → "Allocations"
// 3. Sort by "Allocation in New TLAB"
// 4. Look for:
//    - High allocation rate classes
//    - Unexpected allocations
//    - Allocations in hot loops

// Programmatic leak detection
public class LeakDetector {
    
    public void detectLeaks() throws Exception {
        Configuration config = Configuration.getConfiguration("profile");
        
        try (Recording recording = new Recording(config)) {
            recording.enable("jdk.ObjectAllocationSample")
                     .withStackTrace();
            
            recording.start();
            
            // Run application for a while
            Thread.sleep(300_000);
            
            recording.stop();
            
            // Analyze allocations
            Path recordingFile = Files.createTempFile("alloc", ".jfr");
            recording.dump(recordingFile);
            
            analyzeAllocations(recordingFile);
        }
    }
    
    private void analyzeAllocations(Path recordingFile) throws Exception {
        Map<String, Long> allocations = new HashMap<>();
        
        try (RecordingFile rf = new RecordingFile(recordingFile)) {
            while (rf.hasMoreEvents()) {
                RecordedEvent event = rf.readEvent();
                
                if (event.getEventType().getName().equals("jdk.ObjectAllocationSample")) {
                    RecordedClass clazz = event.getClass("objectClass");
                    String className = clazz.getName();
                    long weight = event.getLong("weight");
                    
                    allocations.merge(className, weight, Long::sum);
                }
            }
        }
        
        // Print top allocators
        allocations.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(20)
            .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
    }
}
```

---

## CPU Profiling

### 1. **Finding Hot Methods with JFR**

```bash
# Profile CPU with JFR
jcmd <PID> JFR.start name=cpu-profile settings=profile duration=60s

# Wait and dump
jcmd <PID> JFR.dump name=cpu-profile filename=cpu.jfr

# Analyze in JMC
jmc cpu.jfr

# Look at:
# 1. Method Profiling tab
# 2. Hot Methods (sorted by self time)
# 3. Hot Packages
# 4. Call Tree (to see call hierarchy)
```

### 2. **CPU Profiling with Async Profiler**

```bash
# Profile CPU for 60 seconds
./profiler.sh -d 60 -f cpu-flamegraph.html <PID>

# Profile specific package
./profiler.sh -d 60 --include 'com.example.*' -f cpu.html <PID>

# Profile with higher sampling rate
./profiler.sh -d 60 -i 1ms -f cpu.html <PID>

# Generate multiple formats
./profiler.sh -d 60 -o collapsed -f cpu.collapsed <PID>
./profiler.sh -d 60 -o flamegraph -f cpu.svg <PID>
./profiler.sh -d 60 -o tree -f cpu.html <PID>
```

### 3. **Thread Profiling**

```bash
# Thread dump
jstack <PID> > thread-dump.txt

# Multiple thread dumps (to detect stuck threads)
for i in {1..5}; do
  jstack <PID> > thread-dump-$i.txt
  sleep 5
done

# Analyze with FastThread.io
# Upload thread dumps to https://fastthread.io

# Look for:
# - BLOCKED threads
# - WAITING threads
# - Threads with same stack trace across dumps (stuck!)
# - Lock contention

# Thread dump analysis script
#!/bin/bash
DUMPS="thread-dump-*.txt"

echo "=== Thread State Summary ==="
for dump in $DUMPS; do
  echo "File: $dump"
  echo "RUNNABLE: $(grep -c "RUNNABLE" $dump)"
  echo "BLOCKED: $(grep -c "BLOCKED" $dump)"
  echo "WAITING: $(grep -c "WAITING" $dump)"
  echo "TIMED_WAITING: $(grep -c "TIMED_WAITING" $dump)"
  echo ""
done

echo "=== Most Common Stack Traces ==="
for dump in $DUMPS; do
  grep "at " $dump | sort | uniq -c | sort -rn | head -10
done
```

---

## I/O and Lock Profiling

### 1. **I/O Profiling with JFR**

```bash
# Enable I/O events
jcmd <PID> JFR.start \
  name=io-profile \
  settings=profile \
  duration=60s \
  filename=io-profile.jfr

# Custom settings for I/O focus
cat > io-profile.jfc << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <event name="jdk.FileRead">
    <setting name="enabled">true</setting>
    <setting name="threshold">1 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <event name="jdk.FileWrite">
    <setting name="enabled">true</setting>
    <setting name="threshold">1 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <event name="jdk.SocketRead">
    <setting name="enabled">true</setting>
    <setting name="threshold">1 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
  
  <event name="jdk.SocketWrite">
    <setting name="enabled">true</setting>
    <setting name="threshold">1 ms</setting>
    <setting name="stackTrace">true</setting>
  </event>
</configuration>
EOF

jcmd <PID> JFR.start settings=io-profile.jfc filename=io.jfr duration=60s
```

### 2. **Lock Profiling with Async Profiler**

```bash
# Profile lock contention
./profiler.sh -e lock -d 60 -f lock-profile.html <PID>

# Wall-clock profiling (shows where threads spend time, including waiting)
./profiler.sh -e wall -d 60 -f wall-profile.html <PID>

# Compare CPU vs Wall (difference shows I/O and blocking)
./profiler.sh -e cpu -d 60 -f cpu.html <PID>
./profiler.sh -e wall -d 60 -f wall.html <PID>

# If wall >> cpu, application is I/O or lock bound
```

### 3. **Detecting Deadlocks**

```bash
# Detect deadlocks
jcmd <PID> Thread.print | grep -A 20 "Found one Java-level deadlock"

# Or with jstack
jstack <PID> | grep -A 20 "Found one Java-level deadlock"

# Programmatic deadlock detection
import java.lang.management.*;

public class DeadlockDetector {
    
    public void detectDeadlocks() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        
        long[] deadlockedThreads = threadBean.findDeadlockedThreads();
        
        if (deadlockedThreads != null) {
            System.out.println("DEADLOCK DETECTED!");
            
            ThreadInfo[] threadInfos = threadBean.getThreadInfo(deadlockedThreads);
            
            for (ThreadInfo threadInfo : threadInfos) {
                System.out.println("Thread: " + threadInfo.getThreadName());
                System.out.println("State: " + threadInfo.getThreadState());
                System.out.println("Locked: " + threadInfo.getLockName());
                System.out.println("Waiting for: " + threadInfo.getLockInfo());
                System.out.println("Stack trace:");
                
                for (StackTraceElement element : threadInfo.getStackTrace()) {
                    System.out.println("  " + element);
                }
                
                System.out.println();
            }
        } else {
            System.out.println("No deadlocks detected");
        }
    }
    
    // Monitor for deadlocks continuously
    public void startDeadlockMonitor() {
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        
        executor.scheduleAtFixedRate(() -> {
            detectDeadlocks();
        }, 0, 30, TimeUnit.SECONDS);
    }
}
```

---

## Production Profiling

### 1. **Low-Overhead Production Profiling**

```bash
# JFR with minimal overhead (< 1%)
java \
  -XX:StartFlightRecording=\
settings=default,\
name=production,\
disk=true,\
maxage=6h,\
maxsize=250M,\
dumponexit=true,\
filename=/var/log/jfr/production-%t.jfr \
  MyApp

# Continuous profiling with rotation
java \
  -XX:StartFlightRecording=\
settings=default,\
name=continuous,\
disk=true,\
dumponexit=true,\
duration=1h,\
filename=/var/log/jfr/app-%t.jfr \
  MyApp

# Profile on-demand in production
jcmd <PID> JFR.start name=prod-profile settings=default duration=120s filename=/tmp/prod.jfr
```

### 2. **Production-Safe Async Profiler**

```bash
# CPU profiling with low overhead
./profiler.sh -e cpu -i 10ms -d 60 -f /tmp/prod-cpu.html <PID>

# Safe allocation profiling
./profiler.sh -e alloc -d 60 --total -f /tmp/prod-alloc.html <PID>

# Schedule periodic profiling
*/15 * * * * /path/to/profiler.sh -e cpu -d 60 -f /var/log/profile-$(date +\%Y\%m\%d-\%H\%M).html $(pgrep -f MyApp)
```

### 3. **Automated Profiling on High CPU**

```bash
#!/bin/bash
# auto-profile-on-high-cpu.sh

THRESHOLD=80
PID=$(pgrep -f MyApp)
PROFILE_DIR="/var/log/profiles"

while true; do
  CPU=$(ps -p $PID -o %cpu= | awk '{print int($1)}')
  
  if [ $CPU -gt $THRESHOLD ]; then
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    echo "High CPU detected: $CPU%. Starting profile..."
    
    # Profile for 60 seconds
    /path/to/profiler.sh \
      -e cpu \
      -d 60 \
      -f $PROFILE_DIR/high-cpu-$TIMESTAMP.html \
      $PID
    
    # Also capture thread dump
    jstack $PID > $PROFILE_DIR/thread-dump-$TIMESTAMP.txt
    
    echo "Profile saved to $PROFILE_DIR/high-cpu-$TIMESTAMP.html"
    
    # Wait before next check to avoid duplicate profiles
    sleep 300
  fi
  
  sleep 10
done
```

---

## Performance Optimization Workflow

### 1. **Systematic Performance Investigation**

```
Step 1: Identify the Problem
├─ Symptom: High CPU? High memory? Slow response?
├─ Metrics: Collect baseline metrics
│   ├─ CPU usage
│   ├─ Memory usage
│   ├─ GC frequency/duration
│   ├─ Response times
│   └─ Throughput
└─ Reproduce: Can you reproduce the issue?

Step 2: Profile the Application
├─ CPU profiling
│   ├─ JFR or Async Profiler
│   ├─ Find hot methods
│   └─ Generate flame graphs
├─ Memory profiling
│   ├─ JFR allocation profiling
│   ├─ Heap dumps
│   └─ Check for leaks
├─ GC analysis
│   ├─ Enable GC logging
│   ├─ Analyze patterns
│   └─ Check pause times
└─ I/O profiling
    ├─ JFR I/O events
    ├─ Database query times
    └─ Network latency

Step 3: Analyze Results
├─ Identify bottlenecks
│   ├─ CPU: Hot methods
│   ├─ Memory: High allocators
│   ├─ GC: Pause times
│   └─ I/O: Slow operations
├─ Prioritize by impact
└─ Form hypothesis

Step 4: Optimize
├─ Make targeted changes
├─ One change at a time
└─ Measure impact

Step 5: Verify
├─ Profile again
├─ Compare before/after
├─ Check metrics improved
└─ No regressions?

Step 6: Deploy and Monitor
├─ Deploy to production
├─ Monitor metrics
└─ Continuous profiling
```

### 2. **Common Optimization Patterns**

```java
// CPU Optimization Example

// ❌ BEFORE: Hot method from profiling
public List<Order> findActiveOrders() {
    return orders.stream()
        .filter(order -> isActive(order))  // Hot: called millions of times
        .collect(Collectors.toList());
}

private boolean isActive(Order order) {
    // Expensive computation
    return order.getStatus() == OrderStatus.ACTIVE &&
           order.getExpiryDate().isAfter(LocalDateTime.now());
}

// ✅ AFTER: Cache the 'now' calculation
public List<Order> findActiveOrders() {
    LocalDateTime now = LocalDateTime.now();  // Call once
    return orders.stream()
        .filter(order -> isActive(order, now))
        .collect(Collectors.toList());
}

private boolean isActive(Order order, LocalDateTime now) {
    return order.getStatus() == OrderStatus.ACTIVE &&
           order.getExpiryDate().isAfter(now);
}

// Memory Optimization Example

// ❌ BEFORE: High allocation rate from profiling
public String processData(List<String> items) {
    String result = "";
    for (String item : items) {
        result += item + ",";  // Creates new String each iteration!
    }
    return result;
}

// ✅ AFTER: Use StringBuilder
public String processData(List<String> items) {
    StringBuilder result = new StringBuilder(items.size() * 20);
    for (String item : items) {
        result.append(item).append(",");
    }
    return result.toString();
}

// GC Optimization Example

// ❌ BEFORE: Promotes objects to Old Gen
public class SessionManager {
    private Map<String, Session> sessions = new HashMap<>();
    
    public void processSession(String sessionId) {
        Session session = sessions.get(sessionId);
        // Long-lived session objects
    }
}

// ✅ AFTER: Use soft references for cache
public class SessionManager {
    private Map<String, SoftReference<Session>> sessions = new HashMap<>();
    
    public void processSession(String sessionId) {
        SoftReference<Session> ref = sessions.get(sessionId);
        Session session = ref != null ? ref.get() : null;
        
        if (session == null) {
            session = loadSession(sessionId);
            sessions.put(sessionId, new SoftReference<>(session));
        }
    }
}
```

---

## Tools Comparison

```
┌─────────────────┬────────────┬───────────┬─────────────┬──────────────┐
│ Tool            │ Overhead   │ Use Case  │ Pros        │ Cons         │
├─────────────────┼────────────┼───────────┼─────────────┼──────────────┤
│ JFR             │ Very Low   │ Production│ Comprehensive│ Complex UI  │
│                 │ (<1%)      │ Profiling │ Low overhead│              │
│                 │            │           │ Many events │              │
├─────────────────┼────────────┼───────────┼─────────────┼──────────────┤
│ Async Profiler │ Very Low   │ Production│ Great flame │ Linux/Mac    │
│                 │ (<1%)      │ CPU/Alloc │ graphs      │ only         │
│                 │            │           │ Easy to use │              │
├─────────────────┼────────────┼───────────┼─────────────┼──────────────┤
│ JProfiler       │ Medium     │ Dev/Test  │ Great UI    │ Commercial   │
│                 │ (5-10%)    │           │ All-in-one  │ High overhead│
├─────────────────┼────────────┼───────────┼─────────────┼──────────────┤
│ YourKit        │ Medium     │ Dev/Test  │ Great UI    │ Commercial   │
│                 │ (5-10%)    │           │ Good docs   │ High overhead│
├─────────────────┼────────────┼───────────┼─────────────┼──────────────┤
│ VisualVM        │ Medium     │ Dev/Test  │ Free        │ Basic        │
│                 │ (5-15%)    │           │ Built-in    │ Limited      │
└─────────────────┴────────────┴───────────┴─────────────┴──────────────┘

Recommendation:
├─ Production: JFR + Async Profiler
├─ Development: JFR + JMC or commercial profiler
├─ Quick checks: Async Profiler, jstack, jmap
└─ Memory leaks: JFR + MAT/JProfiler
```

---

## Best Practices

### ✅ DO

1. **Profile in production** - Use JFR/Async Profiler (low overhead)
2. **Profile continuously** - Regular profiling catches regressions early
3. **Use flame graphs** - Visual representation of hot paths
4. **Analyze GC logs** - Understand heap behavior
5. **Enable JFR by default** - Minimal overhead, huge value
6. **Take heap dumps wisely** - Only when needed, impacts performance
7. **Compare before/after** - Measure optimization impact
8. **Profile representative load** - Real workload, not synthetic
9. **Monitor metrics** - CPU, memory, GC alongside profiling
10. **Automate profiling** - Trigger on high CPU/memory

### ❌ DON'T

1. **Don't profile without a goal** - Know what you're looking for
2. **Don't optimize prematurely** - Profile first, then optimize
3. **Don't trust microbenchmarks** - JIT, GC make them unreliable
4. **Don't profile in debug mode** - Use production-like settings
5. **Don't ignore GC** - GC pauses affect overall performance
6. **Don't forget warm-up** - JIT needs time to optimize
7. **Don't over-sample** - Balance accuracy vs overhead
8. **Don't profile just once** - Variance matters, profile multiple times
9. **Don't ignore I/O** - Often the real bottleneck
10. **Don't guess** - Measure, don't assume

---

## Conclusion

**Essential Profiling Toolkit:**

1. **JFR** - Comprehensive, low-overhead, production-ready
2. **Async Profiler** - Fast CPU/allocation profiling, flame graphs
3. **GC Logs** - Understand heap behavior and GC impact
4. **Heap Dumps** - Find memory leaks (MAT/JProfiler)
5. **Thread Dumps** - Deadlocks, lock contention

**Profiling Workflow:**

```
1. Identify → What's slow?
2. Measure → Baseline metrics
3. Profile → JFR/Async Profiler
4. Analyze → Find bottlenecks
5. Optimize → Fix the issue
6. Verify → Profile again
7. Deploy → Monitor in production
```

**Key Metrics:**

- **CPU**: Hot methods, thread states
- **Memory**: Allocation rate, heap usage, GC frequency
- **GC**: Pause times, frequency, throughput
- **I/O**: Database queries, network calls, file operations
- **Locks**: Contention, blocking time

**Remember**: Performance optimization is an iterative process. Profile, measure, optimize, verify, repeat!
