# Effective Modern Apache Flink 2.0

A comprehensive guide to mastering Apache Flink 2.0 for stream and batch processing, state management, windowing, and building production-grade data pipelines.

---

## Table of Contents

1. [Flink 2.0 Overview](#flink-20-overview)
2. [DataStream API](#datastream-api)
3. [Windowing](#windowing)
4. [State Management](#state-management)
5. [Checkpointing and Fault Tolerance](#checkpointing-and-fault-tolerance)
6. [Watermarks and Event Time](#watermarks-and-event-time)
7. [Table API and SQL](#table-api-and-sql)
8. [Connectors](#connectors)
9. [Side Outputs](#side-outputs)
10. [Broadcast State](#broadcast-state)
11. [Exactly-Once Semantics](#exactly-once-semantics)
12. [Performance Optimization](#performance-optimization)
13. [Real-World Patterns](#real-world-patterns)
14. [Best Practices](#best-practices)

---

## Flink 2.0 Overview

### What's New in Flink 2.0

```java
/**
 * Apache Flink 2.0 Key Features:
 * 
 * 1. Unified Batch and Streaming
 *    - Single API for both modes
 *    - Adaptive execution
 * 
 * 2. SQL Improvements
 *    - Better windowing syntax
 *    - Enhanced time travel queries
 *    - Improved CDC support
 * 
 * 3. State Management
 *    - RocksDB improvements
 *    - State TTL enhancements
 *    - Better rescaling
 * 
 * 4. Checkpointing
 *    - Faster checkpoints
 *    - Unaligned checkpoints by default
 * 
 * 5. Python Support
 *    - PyFlink improvements
 *    - Better UDF support
 * 
 * Maven Dependency:
 * <dependency>
 *   <groupId>org.apache.flink</groupId>
 *   <artifactId>flink-streaming-java</artifactId>
 *   <version>2.0.0</version>
 * </dependency>
 */
```

### Basic Setup

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;

public class FlinkBasicSetup {
    
    public static void main(String[] args) throws Exception {
        // Create execution environment
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Configure parallelism
        env.setParallelism(4);
        
        // Configure checkpointing
        env.enableCheckpointing(60000);  // Every 60 seconds
        
        // Create a data stream
        DataStream<String> stream = env.fromElements(
            "Hello", "World", "Flink", "2.0"
        );
        
        // Process
        stream.map(String::toUpperCase)
              .print();
        
        // Execute
        env.execute("Flink Basic Example");
    }
}
```

---

## DataStream API

### Creating DataStreams

```java
public class DataStreamCreation {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        // From collection
        DataStream<Integer> numbers = env.fromElements(1, 2, 3, 4, 5);
        
        // From collection
        List<String> list = Arrays.asList("a", "b", "c");
        DataStream<String> strings = env.fromCollection(list);
        
        // From socket
        DataStream<String> socketStream = env.socketTextStream("localhost", 9999);
        
        // From Kafka
        KafkaSource<String> kafkaSource = KafkaSource.<String>builder()
            .setBootstrapServers("localhost:9092")
            .setTopics("events")
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();
        
        DataStream<String> kafkaStream = env.fromSource(
            kafkaSource,
            WatermarkStrategy.noWatermarks(),
            "Kafka Source"
        );
        
        // From file
        DataStream<String> fileStream = env.readTextFile("path/to/file.txt");
        
        // Custom source
        DataStream<Event> customStream = env.addSource(new CustomSourceFunction());
        
        env.execute("DataStream Creation");
    }
}
```

### Transformations

```java
public class DataStreamTransformations {
    
    public void transformations(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource());
        
        // Map - one-to-one transformation
        DataStream<String> mapped = events.map(event -> event.getName());
        
        // FlatMap - one-to-many transformation
        DataStream<String> flatMapped = events.flatMap(
            (Event event, Collector<String> out) -> {
                for (String tag : event.getTags()) {
                    out.collect(tag);
                }
            }
        ).returns(String.class);
        
        // Filter
        DataStream<Event> filtered = events.filter(event -> event.getValue() > 100);
        
        // KeyBy - partition by key
        KeyedStream<Event, String> keyed = events.keyBy(Event::getUserId);
        
        // Reduce - aggregation
        DataStream<Event> reduced = keyed.reduce((e1, e2) -> {
            e1.setValue(e1.getValue() + e2.getValue());
            return e1;
        });
        
        // Aggregate
        DataStream<Long> aggregated = keyed.aggregate(new AverageAggregate());
        
        // Process - low-level processing
        DataStream<String> processed = keyed.process(
            new KeyedProcessFunction<String, Event, String>() {
                @Override
                public void processElement(
                        Event event,
                        Context ctx,
                        Collector<String> out) {
                    
                    // Access state
                    // Access timers
                    // Custom logic
                    out.collect(event.toString());
                }
            }
        );
    }
}

class AverageAggregate implements AggregateFunction<Event, Tuple2<Long, Long>, Double> {
    
    @Override
    public Tuple2<Long, Long> createAccumulator() {
        return Tuple2.of(0L, 0L);
    }
    
    @Override
    public Tuple2<Long, Long> add(Event event, Tuple2<Long, Long> acc) {
        return Tuple2.of(acc.f0 + event.getValue(), acc.f1 + 1);
    }
    
    @Override
    public Double getResult(Tuple2<Long, Long> acc) {
        return acc.f1 == 0 ? 0.0 : (double) acc.f0 / acc.f1;
    }
    
    @Override
    public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
        return Tuple2.of(a.f0 + b.f0, a.f1 + b.f1);
    }
}
```

---

## Windowing

### Time Windows

```java
public class TimeWindows {
    
    public void windowExamples(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource());
        KeyedStream<Event, String> keyed = events.keyBy(Event::getUserId);
        
        // Tumbling Window - fixed size, non-overlapping
        DataStream<Long> tumbling = keyed
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new CountAggregate());
        
        // Sliding Window - fixed size, overlapping
        DataStream<Long> sliding = keyed
            .window(SlidingEventTimeWindows.of(
                Time.minutes(10),  // window size
                Time.minutes(5)    // slide interval
            ))
            .aggregate(new CountAggregate());
        
        // Session Window - dynamic size based on inactivity gap
        DataStream<Long> session = keyed
            .window(EventTimeSessionWindows.withGap(Time.minutes(15)))
            .aggregate(new CountAggregate());
        
        // Processing Time Windows
        DataStream<Long> processingTime = keyed
            .window(TumblingProcessingTimeWindows.of(Time.minutes(5)))
            .aggregate(new CountAggregate());
    }
}
```

### Window Functions

```java
public class WindowFunctions {
    
    // Window Aggregate Function
    public static class SumAggregateFunction 
            implements AggregateFunction<Event, Long, Long> {
        
        @Override
        public Long createAccumulator() {
            return 0L;
        }
        
        @Override
        public Long add(Event event, Long accumulator) {
            return accumulator + event.getValue();
        }
        
        @Override
        public Long getResult(Long accumulator) {
            return accumulator;
        }
        
        @Override
        public Long merge(Long a, Long b) {
            return a + b;
        }
    }
    
    // Window Process Function
    public static class WindowProcessFunction extends
            ProcessWindowFunction<Event, String, String, TimeWindow> {
        
        @Override
        public void process(
                String key,
                Context context,
                Iterable<Event> elements,
                Collector<String> out) {
            
            long count = 0;
            for (Event event : elements) {
                count++;
            }
            
            long windowStart = context.window().getStart();
            long windowEnd = context.window().getEnd();
            
            out.collect(String.format(
                "Window[%d-%d]: Key=%s, Count=%d",
                windowStart, windowEnd, key, count
            ));
        }
    }
    
    // Combine Aggregate + Process for efficiency
    public void efficientWindowing(KeyedStream<Event, String> keyed) {
        DataStream<String> result = keyed
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(
                new SumAggregateFunction(),
                new ProcessWindowFunction<Long, String, String, TimeWindow>() {
                    @Override
                    public void process(
                            String key,
                            Context context,
                            Iterable<Long> elements,
                            Collector<String> out) {
                        
                        Long sum = elements.iterator().next();
                        out.collect(String.format(
                            "Key=%s, Sum=%d, Window=%d-%d",
                            key, sum,
                            context.window().getStart(),
                            context.window().getEnd()
                        ));
                    }
                }
            );
    }
}
```

### Custom Windows

```java
public class CustomWindows {
    
    // Count-based window
    public void countWindow(KeyedStream<Event, String> keyed) {
        DataStream<Long> result = keyed
            .countWindow(100)  // Every 100 elements
            .aggregate(new CountAggregate());
    }
    
    // Global window with custom trigger
    public void globalWindowWithTrigger(KeyedStream<Event, String> keyed) {
        DataStream<Long> result = keyed
            .window(GlobalWindows.create())
            .trigger(new CustomTrigger())
            .aggregate(new CountAggregate());
    }
}

class CustomTrigger extends Trigger<Event, GlobalWindow> {
    
    @Override
    public TriggerResult onElement(
            Event element,
            long timestamp,
            GlobalWindow window,
            TriggerContext ctx) {
        
        // Fire every 100 elements
        ValueState<Long> count = ctx.getPartitionedState(
            new ValueStateDescriptor<>("count", Long.class)
        );
        
        Long current = count.value();
        if (current == null) current = 0L;
        current++;
        count.update(current);
        
        if (current >= 100) {
            count.clear();
            return TriggerResult.FIRE_AND_PURGE;
        }
        
        return TriggerResult.CONTINUE;
    }
    
    @Override
    public TriggerResult onProcessingTime(
            long time,
            GlobalWindow window,
            TriggerContext ctx) {
        return TriggerResult.CONTINUE;
    }
    
    @Override
    public TriggerResult onEventTime(
            long time,
            GlobalWindow window,
            TriggerContext ctx) {
        return TriggerResult.CONTINUE;
    }
    
    @Override
    public void clear(GlobalWindow window, TriggerContext ctx) {
        ctx.getPartitionedState(
            new ValueStateDescriptor<>("count", Long.class)
        ).clear();
    }
}
```

---

## State Management

### Keyed State

```java
public class KeyedStateExample extends RichFlatMapFunction<Event, Alert> {
    
    // Value State - single value per key
    private transient ValueState<Long> countState;
    
    // List State - list of values per key
    private transient ListState<String> historyState;
    
    // Map State - map per key
    private transient MapState<String, Long> mapState;
    
    // Reducing State - incrementally aggregated value
    private transient ReducingState<Long> reducingState;
    
    // Aggregating State - custom aggregation
    private transient AggregatingState<Event, Double> aggregatingState;
    
    @Override
    public void open(Configuration parameters) {
        // Initialize ValueState
        ValueStateDescriptor<Long> countDescriptor = 
            new ValueStateDescriptor<>("count", Long.class);
        countState = getRuntimeContext().getState(countDescriptor);
        
        // Initialize ListState
        ListStateDescriptor<String> historyDescriptor = 
            new ListStateDescriptor<>("history", String.class);
        historyState = getRuntimeContext().getListState(historyDescriptor);
        
        // Initialize MapState
        MapStateDescriptor<String, Long> mapDescriptor = 
            new MapStateDescriptor<>("map", String.class, Long.class);
        mapState = getRuntimeContext().getMapState(mapDescriptor);
        
        // Initialize ReducingState
        ReducingStateDescriptor<Long> reducingDescriptor = 
            new ReducingStateDescriptor<>("reducing", Long::sum, Long.class);
        reducingState = getRuntimeContext().getReducingState(reducingDescriptor);
        
        // Initialize AggregatingState
        AggregatingStateDescriptor<Event, Tuple2<Long, Long>, Double> aggDescriptor =
            new AggregatingStateDescriptor<>(
                "aggregating",
                new AverageAggregate(),
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})
            );
        aggregatingState = getRuntimeContext().getAggregatingState(aggDescriptor);
    }
    
    @Override
    public void flatMap(Event event, Collector<Alert> out) throws Exception {
        // ValueState - read and update
        Long count = countState.value();
        if (count == null) count = 0L;
        count++;
        countState.update(count);
        
        // ListState - append
        historyState.add(event.getName());
        
        // MapState - put
        mapState.put(event.getType(), event.getValue());
        
        // ReducingState - add
        reducingState.add(event.getValue());
        
        // Check threshold
        if (count > 100) {
            out.collect(new Alert("High count: " + count));
        }
    }
}
```

### State TTL (Time-To-Live)

```java
public class StateTTLExample extends RichFlatMapFunction<Event, String> {
    
    private transient ValueState<Long> countState;
    
    @Override
    public void open(Configuration parameters) {
        // Configure TTL
        StateTtlConfig ttlConfig = StateTtlConfig
            .newBuilder(Time.hours(1))  // TTL of 1 hour
            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
            .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
            .build();
        
        ValueStateDescriptor<Long> descriptor = 
            new ValueStateDescriptor<>("count", Long.class);
        
        // Enable TTL
        descriptor.enableTimeToLive(ttlConfig);
        
        countState = getRuntimeContext().getState(descriptor);
    }
    
    @Override
    public void flatMap(Event event, Collector<String> out) throws Exception {
        Long count = countState.value();
        if (count == null) count = 0L;
        count++;
        countState.update(count);
        
        out.collect("Count: " + count);
    }
}
```

### Queryable State

```java
public class QueryableStateExample {
    
    public void createQueryableState(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource());
        
        events
            .keyBy(Event::getUserId)
            .flatMap(new RichFlatMapFunction<Event, Void>() {
                
                private transient ValueState<Long> state;
                
                @Override
                public void open(Configuration parameters) {
                    ValueStateDescriptor<Long> descriptor = 
                        new ValueStateDescriptor<>("count", Long.class);
                    
                    // Make state queryable
                    descriptor.setQueryable("user-counts");
                    
                    state = getRuntimeContext().getState(descriptor);
                }
                
                @Override
                public void flatMap(Event event, Collector<Void> out) throws Exception {
                    Long count = state.value();
                    if (count == null) count = 0L;
                    count++;
                    state.update(count);
                }
            });
    }
    
    // Query state from external application
    public void queryState() throws Exception {
        QueryableStateClient client = new QueryableStateClient(
            "localhost",
            9069  // Queryable state port
        );
        
        CompletableFuture<ValueState<Long>> future = client.getKvState(
            JobID.fromHexString("job-id"),
            "user-counts",
            "user-123",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ValueStateDescriptor<>("count", Long.class)
        );
        
        ValueState<Long> state = future.get();
        Long count = state.value();
        System.out.println("User count: " + count);
    }
}
```

---

## Checkpointing and Fault Tolerance

### Checkpoint Configuration

```java
public class CheckpointConfiguration {
    
    public void configureCheckpointing(StreamExecutionEnvironment env) {
        // Enable checkpointing every 60 seconds
        env.enableCheckpointing(60000);
        
        CheckpointConfig config = env.getCheckpointConfig();
        
        // Set checkpointing mode
        config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        
        // Min pause between checkpoints
        config.setMinPauseBetweenCheckpoints(30000);
        
        // Checkpoint timeout
        config.setCheckpointTimeout(60000);
        
        // Max concurrent checkpoints
        config.setMaxConcurrentCheckpoints(1);
        
        // Enable unaligned checkpoints (Flink 2.0 default)
        config.enableUnalignedCheckpoints(true);
        
        // Externalized checkpoints (survive job cancellation)
        config.setExternalizedCheckpointCleanup(
            ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION
        );
        
        // Checkpoint storage
        config.setCheckpointStorage("hdfs://namenode:9000/flink/checkpoints");
        
        // State backend
        env.setStateBackend(new HashMapStateBackend());
        // Or: env.setStateBackend(new EmbeddedRocksDBStateBackend());
    }
}
```

### Recovery and Restart Strategies

```java
public class RestartStrategies {
    
    public void configureRestartStrategy(StreamExecutionEnvironment env) {
        // Fixed delay restart
        env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
            3,              // Number of restart attempts
            Time.seconds(10) // Delay between attempts
        ));
        
        // Exponential delay restart
        env.setRestartStrategy(RestartStrategies.exponentialDelayRestart(
            Time.milliseconds(1),    // Initial backoff
            Time.milliseconds(1000), // Max backoff
            1.5,                     // Backoff multiplier
            Time.milliseconds(10000), // Reset after
            0.1                      // Jitter factor
        ));
        
        // Failure rate restart
        env.setRestartStrategy(RestartStrategies.failureRateRestart(
            3,                // Max failures per interval
            Time.minutes(5),  // Failure rate interval
            Time.seconds(10)  // Delay between attempts
        ));
        
        // No restart
        env.setRestartStrategy(RestartStrategies.noRestart());
    }
}
```

---

## Watermarks and Event Time

### Watermark Strategies

```java
public class WatermarkStrategies {
    
    public void watermarkExamples(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource());
        
        // Monotonous timestamps (no out-of-order)
        DataStream<Event> monotonous = events.assignTimestampsAndWatermarks(
            WatermarkStrategy.<Event>forMonotonousTimestamps()
                .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
        );
        
        // Bounded out-of-orderness
        DataStream<Event> bounded = events.assignTimestampsAndWatermarks(
            WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
        );
        
        // Custom watermark generator
        DataStream<Event> custom = events.assignTimestampsAndWatermarks(
            WatermarkStrategy.forGenerator(ctx -> new CustomWatermarkGenerator())
                .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
        );
        
        // With idleness detection
        DataStream<Event> withIdleness = events.assignTimestampsAndWatermarks(
            WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
                .withIdleness(Duration.ofMinutes(1))
        );
    }
}

class CustomWatermarkGenerator implements WatermarkGenerator<Event> {
    
    private long maxTimestamp = Long.MIN_VALUE;
    private final long outOfOrdernessMillis = 5000;
    
    @Override
    public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
        maxTimestamp = Math.max(maxTimestamp, event.getTimestamp());
    }
    
    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis));
    }
}
```

### Late Data Handling

```java
public class LateDataHandling {
    
    public void handleLateData(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource())
            .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                    .withTimestampAssigner((event, ts) -> event.getTimestamp())
            );
        
        OutputTag<Event> lateOutputTag = new OutputTag<Event>("late-data"){};
        
        SingleOutputStreamOperator<String> result = events
            .keyBy(Event::getUserId)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .allowedLateness(Time.minutes(1))  // Allow 1 minute late
            .sideOutputLateData(lateOutputTag) // Send late data to side output
            .aggregate(new CountAggregate(),
                new ProcessWindowFunction<Long, String, String, TimeWindow>() {
                    @Override
                    public void process(
                            String key,
                            Context context,
                            Iterable<Long> elements,
                            Collector<String> out) {
                        out.collect("Window result: " + elements.iterator().next());
                    }
                }
            );
        
        // Process late data
        DataStream<Event> lateData = result.getSideOutput(lateOutputTag);
        lateData.print("Late Data");
    }
}
```

---

## Table API and SQL

### Table API

```java
public class TableAPIExample {
    
    public void tableAPIExample() {
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        StreamTableEnvironment tableEnv = 
            StreamTableEnvironment.create(env);
        
        // Create table from DataStream
        DataStream<Event> events = env.addSource(new EventSource());
        Table eventTable = tableEnv.fromDataStream(events);
        
        // Table API operations
        Table result = eventTable
            .filter($("value").isGreater(100))
            .groupBy($("userId"))
            .select(
                $("userId"),
                $("value").sum().as("total"),
                $("value").avg().as("average")
            );
        
        // Convert back to DataStream
        DataStream<Row> resultStream = tableEnv.toDataStream(result);
        resultStream.print();
        
        env.execute("Table API Example");
    }
}
```

### Flink SQL

```java
public class FlinkSQLExample {
    
    public void sqlExample() {
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        StreamTableEnvironment tableEnv = 
            StreamTableEnvironment.create(env);
        
        // Register table
        tableEnv.executeSql(
            "CREATE TABLE events (" +
            "  user_id STRING," +
            "  event_type STRING," +
            "  value BIGINT," +
            "  event_time TIMESTAMP(3)," +
            "  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND" +
            ") WITH (" +
            "  'connector' = 'kafka'," +
            "  'topic' = 'events'," +
            "  'properties.bootstrap.servers' = 'localhost:9092'," +
            "  'format' = 'json'" +
            ")"
        );
        
        // SQL query
        Table result = tableEnv.sqlQuery(
            "SELECT " +
            "  user_id, " +
            "  COUNT(*) as event_count, " +
            "  SUM(value) as total_value " +
            "FROM events " +
            "GROUP BY user_id"
        );
        
        // Windowed aggregation
        Table windowedResult = tableEnv.sqlQuery(
            "SELECT " +
            "  user_id, " +
            "  TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start, " +
            "  COUNT(*) as event_count " +
            "FROM events " +
            "GROUP BY " +
            "  user_id, " +
            "  TUMBLE(event_time, INTERVAL '5' MINUTE)"
        );
        
        // Execute and print
        tableEnv.executeSql("SELECT * FROM " + windowedResult).print();
    }
}
```

---

## Connectors

### Kafka Connector

```java
public class KafkaConnectorExample {
    
    // Kafka Source
    public void kafkaSource(StreamExecutionEnvironment env) {
        KafkaSource<Event> source = KafkaSource.<Event>builder()
            .setBootstrapServers("localhost:9092")
            .setTopics("events")
            .setGroupId("flink-consumer-group")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setValueOnlyDeserializer(new EventDeserializationSchema())
            .build();
        
        DataStream<Event> stream = env.fromSource(
            source,
            WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                .withTimestampAssigner((event, ts) -> event.getTimestamp()),
            "Kafka Source"
        );
    }
    
    // Kafka Sink
    public void kafkaSink(DataStream<Event> stream) {
        KafkaSink<Event> sink = KafkaSink.<Event>builder()
            .setBootstrapServers("localhost:9092")
            .setRecordSerializer(
                KafkaRecordSerializationSchema.builder()
                    .setTopic("output-events")
                    .setValueSerializationSchema(new EventSerializationSchema())
                    .build()
            )
            .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
            .setTransactionalIdPrefix("flink-kafka-sink")
            .build();
        
        stream.sinkTo(sink);
    }
}
```

### JDBC Connector

```java
public class JDBCConnectorExample {
    
    public void jdbcSink(DataStream<Event> stream) {
        stream.addSink(
            JdbcSink.sink(
                "INSERT INTO events (user_id, event_type, value, timestamp) " +
                "VALUES (?, ?, ?, ?)",
                (statement, event) -> {
                    statement.setString(1, event.getUserId());
                    statement.setString(2, event.getEventType());
                    statement.setLong(3, event.getValue());
                    statement.setTimestamp(4, new Timestamp(event.getTimestamp()));
                },
                JdbcExecutionOptions.builder()
                    .withBatchSize(1000)
                    .withBatchIntervalMs(200)
                    .withMaxRetries(5)
                    .build(),
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                    .withUrl("jdbc:postgresql://localhost:5432/mydb")
                    .withDriverName("org.postgresql.Driver")
                    .withUsername("user")
                    .withPassword("password")
                    .build()
            )
        );
    }
}
```

---

## Side Outputs

### Using Side Outputs

```java
public class SideOutputExample {
    
    public void sideOutputExample(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource());
        
        // Define output tags
        OutputTag<Event> highValueTag = new OutputTag<Event>("high-value"){};
        OutputTag<Event> errorTag = new OutputTag<Event>("errors"){};
        
        // Process with side outputs
        SingleOutputStreamOperator<Event> mainStream = events.process(
            new ProcessFunction<Event, Event>() {
                @Override
                public void processElement(
                        Event event,
                        Context ctx,
                        Collector<Event> out) {
                    
                    // Main output - normal events
                    if (event.getValue() <= 1000) {
                        out.collect(event);
                    }
                    // High value events
                    else if (event.getValue() > 1000) {
                        ctx.output(highValueTag, event);
                    }
                    
                    // Error events
                    if (event.getEventType().equals("ERROR")) {
                        ctx.output(errorTag, event);
                    }
                }
            }
        );
        
        // Get side outputs
        DataStream<Event> highValueStream = mainStream.getSideOutput(highValueTag);
        DataStream<Event> errorStream = mainStream.getSideOutput(errorTag);
        
        // Process each stream separately
        mainStream.print("Normal");
        highValueStream.print("High Value");
        errorStream.print("Errors");
    }
}
```

---

## Broadcast State

### Broadcast State Pattern

```java
public class BroadcastStateExample {
    
    public void broadcastStateExample(StreamExecutionEnvironment env) {
        // Configuration stream (broadcast)
        DataStream<Config> configStream = env.addSource(new ConfigSource());
        
        // Event stream
        DataStream<Event> eventStream = env.addSource(new EventSource());
        
        // Define broadcast state descriptor
        MapStateDescriptor<String, Config> configDescriptor = 
            new MapStateDescriptor<>(
                "config",
                String.class,
                Config.class
            );
        
        // Broadcast configuration
        BroadcastStream<Config> broadcastStream = 
            configStream.broadcast(configDescriptor);
        
        // Connect and process
        DataStream<String> result = eventStream
            .keyBy(Event::getUserId)
            .connect(broadcastStream)
            .process(new KeyedBroadcastProcessFunction<
                    String, Event, Config, String>() {
                
                @Override
                public void processElement(
                        Event event,
                        ReadOnlyContext ctx,
                        Collector<String> out) throws Exception {
                    
                    // Read broadcast state
                    ReadOnlyBroadcastState<String, Config> broadcastState = 
                        ctx.getBroadcastState(configDescriptor);
                    
                    Config config = broadcastState.get(event.getEventType());
                    
                    if (config != null && event.getValue() > config.getThreshold()) {
                        out.collect("Alert: " + event);
                    }
                }
                
                @Override
                public void processBroadcastElement(
                        Config config,
                        Context ctx,
                        Collector<String> out) throws Exception {
                    
                    // Update broadcast state
                    BroadcastState<String, Config> broadcastState = 
                        ctx.getBroadcastState(configDescriptor);
                    
                    broadcastState.put(config.getEventType(), config);
                }
            });
        
        result.print();
    }
}
```

---

## Exactly-Once Semantics

### Two-Phase Commit Sink

```java
public class ExactlyOnceSink extends TwoPhaseCommitSinkFunction<Event, Transaction, Void> {
    
    public ExactlyOnceSink() {
        super(
            TypeInformation.of(Transaction.class).createSerializer(null),
            TypeInformation.of(Void.class).createSerializer(null)
        );
    }
    
    @Override
    protected void invoke(
            Transaction transaction,
            Event event,
            Context context) {
        
        // Write to transaction buffer
        transaction.write(event);
    }
    
    @Override
    protected Transaction beginTransaction() {
        // Start new transaction
        return new Transaction();
    }
    
    @Override
    protected void preCommit(Transaction transaction) {
        // Prepare transaction for commit
        transaction.prepare();
    }
    
    @Override
    protected void commit(Transaction transaction) {
        // Commit transaction
        transaction.commit();
    }
    
    @Override
    protected void abort(Transaction transaction) {
        // Rollback transaction
        transaction.rollback();
    }
}
```

---

## Performance Optimization

### Parallelism and Resource Configuration

```java
public class PerformanceOptimization {
    
    public void optimizePerformance(StreamExecutionEnvironment env) {
        // Global parallelism
        env.setParallelism(8);
        
        // Per-operator parallelism
        DataStream<Event> events = env.addSource(new EventSource())
            .setParallelism(4);  // Source parallelism
        
        events
            .map(Event::transform)
            .setParallelism(8)   // Map parallelism
            .keyBy(Event::getUserId)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new SumAggregate())
            .setParallelism(4);  // Window parallelism
        
        // Buffer timeout (trade latency for throughput)
        env.setBufferTimeout(100);  // 100ms
        
        // Disable operator chaining for debugging
        env.disableOperatorChaining();
        
        // Or per-operator
        events.map(Event::transform)
            .disableChaining();
    }
}
```

---

## Real-World Patterns

### Sessionization

```java
public class Sessionization {
    
    public void sessionize(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource())
            .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                    .withTimestampAssigner((event, ts) -> event.getTimestamp())
            );
        
        DataStream<SessionSummary> sessions = events
            .keyBy(Event::getUserId)
            .window(EventTimeSessionWindows.withGap(Time.minutes(30)))
            .process(new SessionProcessor());
        
        sessions.print();
    }
}

class SessionProcessor extends
        ProcessWindowFunction<Event, SessionSummary, String, TimeWindow> {
    
    @Override
    public void process(
            String userId,
            Context context,
            Iterable<Event> events,
            Collector<SessionSummary> out) {
        
        long sessionStart = context.window().getStart();
        long sessionEnd = context.window().getEnd();
        
        List<Event> eventList = new ArrayList<>();
        events.forEach(eventList::add);
        
        SessionSummary summary = new SessionSummary(
            userId,
            sessionStart,
            sessionEnd,
            eventList.size(),
            eventList
        );
        
        out.collect(summary);
    }
}
```

### CEP (Complex Event Processing)

```java
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.cep.pattern.conditions.SimpleCondition;

public class CEPExample {
    
    public void detectPatterns(StreamExecutionEnvironment env) {
        DataStream<Event> events = env.addSource(new EventSource())
            .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                    .withTimestampAssigner((event, ts) -> event.getTimestamp())
            );
        
        // Define pattern: start -> middle -> end within 10 minutes
        Pattern<Event, ?> pattern = Pattern.<Event>begin("start")
            .where(new SimpleCondition<Event>() {
                @Override
                public boolean filter(Event event) {
                    return event.getEventType().equals("LOGIN");
                }
            })
            .next("middle")
            .where(new SimpleCondition<Event>() {
                @Override
                public boolean filter(Event event) {
                    return event.getEventType().equals("PURCHASE");
                }
            })
            .times(2, 5)  // 2 to 5 purchases
            .within(Time.minutes(10));
        
        // Apply pattern
        PatternStream<Event> patternStream = CEP.pattern(
            events.keyBy(Event::getUserId),
            pattern
        );
        
        // Select matched patterns
        DataStream<Alert> alerts = patternStream.select(
            (Map<String, List<Event>> pattern) -> {
                List<Event> starts = pattern.get("start");
                List<Event> middles = pattern.get("middle");
                
                return new Alert(
                    "Pattern matched: " + middles.size() + " purchases after login"
                );
            }
        );
        
        alerts.print();
    }
}
```

---

## Best Practices

### ✅ DO

1. **Use event time for windowing**
2. **Enable checkpointing**
3. **Set parallelism appropriately**
4. **Use RocksDB for large state**
5. **Configure watermarks correctly**
6. **Use side outputs for branching**
7. **Monitor backpressure**
8. **Test with realistic data volumes**
9. **Use broadcast state for configuration**
10. **Enable metrics and logging**

### ❌ DON'T

1. **Don't use processing time unless necessary**
2. **Don't ignore late data**
3. **Don't set parallelism too high**
4. **Don't forget state cleanup (TTL)**
5. **Don't ignore backpressure warnings**

---

## Conclusion

**Key Takeaways:**

1. **DataStream API** - Core streaming abstraction
2. **Windowing** - Tumbling, sliding, session windows
3. **State** - Keyed state with TTL
4. **Checkpointing** - Fault tolerance
5. **Watermarks** - Event time processing
6. **Table API/SQL** - Declarative processing
7. **Connectors** - Kafka, JDBC, file systems
8. **Exactly-Once** - End-to-end guarantees

**Remember**: Flink excels at stateful stream processing with exactly-once guarantees and event-time semantics!
