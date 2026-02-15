# Effective Ways of Using Kafka Serialization and Deserialization

A comprehensive guide to mastering Kafka serialization, schema management, and data evolution.

---

## Table of Contents

1. [Serialization Fundamentals](#serialization-fundamentals)
2. [Built-in Serializers](#built-in-serializers)
3. [JSON Serialization](#json-serialization)
4. [Avro Serialization](#avro-serialization)
5. [Protobuf Serialization](#protobuf-serialization)
6. [Schema Registry](#schema-registry)
7. [Custom Serializers](#custom-serializers)
8. [Error Handling](#error-handling)
9. [Schema Evolution](#schema-evolution)
10. [Performance Optimization](#performance-optimization)
11. [Testing Strategies](#testing-strategies)
12. [Best Practices](#best-practices)
13. [Decision Tree](#decision-tree)

---

## Serialization Fundamentals

### Why Serialization Matters

```java
/**
 * Kafka stores and transmits data as byte arrays
 * 
 * Producer: Object → byte[] → Kafka
 * Consumer: Kafka → byte[] → Object
 * 
 * Key Requirements:
 * 1. Compact format (network efficiency)
 * 2. Fast serialization/deserialization
 * 3. Schema evolution support
 * 4. Cross-language compatibility (optional)
 * 5. Human readability (optional, debugging)
 */

public class SerializationBasics {
    
    // Producer flow
    public void producerFlow() {
        User user = new User("123", "John Doe");
        
        // 1. Application object
        // 2. Serializer converts to byte[]
        // 3. Kafka stores byte[]
        
        producer.send(new ProducerRecord<>("users", user.getId(), user));
    }
    
    // Consumer flow
    public void consumerFlow() {
        ConsumerRecords<String, User> records = consumer.poll(Duration.ofMillis(100));
        
        for (ConsumerRecord<String, User> record : records) {
            // 1. Kafka provides byte[]
            // 2. Deserializer converts to object
            // 3. Application uses object
            
            User user = record.value();
            processUser(user);
        }
    }
}
```

### Serialization Format Comparison

```
┌─────────────────┬──────────┬───────────┬────────────┬──────────────┬─────────────┐
│ Format          │ Size     │ Speed     │ Evolution  │ Readability  │ Cross-Lang  │
├─────────────────┼──────────┼───────────┼────────────┼──────────────┼─────────────┤
│ String/Bytes    │ Varies   │ Fast      │ None       │ Yes (String) │ Yes         │
│ JSON            │ Large    │ Slow      │ Manual     │ Excellent    │ Yes         │
│ Avro            │ Small    │ Fast      │ Excellent  │ No           │ Yes         │
│ Protobuf        │ Smallest │ Fastest   │ Good       │ No           │ Yes         │
│ Thrift          │ Small    │ Fast      │ Good       │ No           │ Yes         │
│ Java Serialize  │ Large    │ Slow      │ Poor       │ No           │ No (Java)   │
└─────────────────┴──────────┴───────────┴────────────┴──────────────┴─────────────┘
```

---

## Built-in Serializers

### 1. **String Serializer**

```java
public class StringSerializerExamples {
    
    // Configuration
    public Properties configureProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        
        return props;
    }
    
    // Producer
    public void produceStringMessages() {
        try (KafkaProducer<String, String> producer = new KafkaProducer<>(configureProducer())) {
            
            ProducerRecord<String, String> record = new ProducerRecord<>(
                "events",
                "user-123",  // Key: String
                "User logged in"  // Value: String
            );
            
            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    logger.error("Failed to send message", exception);
                } else {
                    logger.info("Sent to partition {} at offset {}", 
                        metadata.partition(), metadata.offset());
                }
            });
        }
    }
    
    // Consumer
    public void consumeStringMessages() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("events"));
            
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    logger.info("Key: {}, Value: {}", record.key(), record.value());
                }
            }
        }
    }
}
```

### 2. **Integer, Long, Double Serializers**

```java
public class NumericSerializers {
    
    // Integer key, Long value example
    public void configureNumericProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            IntegerSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            LongSerializer.class.getName());
        
        try (KafkaProducer<Integer, Long> producer = new KafkaProducer<>(props)) {
            
            ProducerRecord<Integer, Long> record = new ProducerRecord<>(
                "metrics",
                42,           // Integer key
                1234567890L   // Long value
            );
            
            producer.send(record);
        }
    }
    
    // Common use case: Counters and metrics
    public class MetricsProducer {
        
        private final KafkaProducer<String, Long> producer;
        
        public MetricsProducer(KafkaProducer<String, Long> producer) {
            this.producer = producer;
        }
        
        public void publishMetric(String metricName, long value) {
            ProducerRecord<String, Long> record = new ProducerRecord<>(
                "metrics",
                metricName,  // Key: metric name
                value        // Value: metric value
            );
            
            producer.send(record);
        }
    }
}
```

### 3. **ByteArray Serializer**

```java
public class ByteArraySerializer {
    
    // Most flexible - you control serialization
    public void produceBinaryData() {
        Properties props = new Properties();
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            ByteArraySerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            ByteArraySerializer.class.getName());
        
        try (KafkaProducer<byte[], byte[]> producer = new KafkaProducer<>(props)) {
            
            // Serialize yourself
            byte[] key = "user-123".getBytes(StandardCharsets.UTF_8);
            byte[] value = serializeUser(new User("123", "John"));
            
            ProducerRecord<byte[], byte[]> record = new ProducerRecord<>(
                "users", key, value
            );
            
            producer.send(record);
        }
    }
    
    private byte[] serializeUser(User user) {
        // Custom serialization logic
        return objectMapper.writeValueAsBytes(user);
    }
}
```

---

## JSON Serialization

### 1. **Jackson-based JSON Serialization**

```java
public class JsonSerializationWithJackson {
    
    // Domain object
    public static class User {
        private String id;
        private String name;
        private String email;
        private Instant createdAt;
        
        // Constructors, getters, setters
    }
    
    // Custom JSON Serializer
    public class JsonSerializer<T> implements Serializer<T> {
        
        private final ObjectMapper objectMapper;
        
        public JsonSerializer() {
            this.objectMapper = new ObjectMapper();
            objectMapper.registerModule(new JavaTimeModule());
            objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        }
        
        @Override
        public void configure(Map<String, ?> configs, boolean isKey) {
            // Configuration if needed
        }
        
        @Override
        public byte[] serialize(String topic, T data) {
            if (data == null) {
                return null;
            }
            
            try {
                return objectMapper.writeValueAsBytes(data);
            } catch (JsonProcessingException e) {
                throw new SerializationException("Error serializing JSON", e);
            }
        }
        
        @Override
        public void close() {
            // Cleanup if needed
        }
    }
    
    // Custom JSON Deserializer
    public class JsonDeserializer<T> implements Deserializer<T> {
        
        private final ObjectMapper objectMapper;
        private final Class<T> targetClass;
        
        public JsonDeserializer(Class<T> targetClass) {
            this.targetClass = targetClass;
            this.objectMapper = new ObjectMapper();
            objectMapper.registerModule(new JavaTimeModule());
        }
        
        @Override
        public void configure(Map<String, ?> configs, boolean isKey) {
            // Configuration if needed
        }
        
        @Override
        public T deserialize(String topic, byte[] data) {
            if (data == null) {
                return null;
            }
            
            try {
                return objectMapper.readValue(data, targetClass);
            } catch (IOException e) {
                throw new SerializationException("Error deserializing JSON", e);
            }
        }
        
        @Override
        public void close() {
            // Cleanup if needed
        }
    }
    
    // Producer configuration
    public KafkaProducer<String, User> createProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            JsonSerializer.class.getName());
        
        return new KafkaProducer<>(props);
    }
    
    // Consumer configuration
    public KafkaConsumer<String, User> createConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "user-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            JsonDeserializer.class.getName());
        
        return new KafkaConsumer<>(props, 
            new StringDeserializer(), 
            new JsonDeserializer<>(User.class));
    }
}
```

### 2. **Spring Kafka JSON Serialization**

```java
@Configuration
public class KafkaJsonConfig {
    
    // Producer configuration
    @Bean
    public ProducerFactory<String, User> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, User> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    // Consumer configuration
    @Bean
    public ConsumerFactory<String, User> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "user-consumer");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        
        // Configure JSON deserializer
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.model");
        config.put(JsonDeserializer.VALUE_DEFAULT_TYPE, User.class.getName());
        
        return new DefaultKafkaConsumerFactory<>(config);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, User> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, User> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}

// Producer service
@Service
public class UserProducer {
    
    private final KafkaTemplate<String, User> kafkaTemplate;
    
    public UserProducer(KafkaTemplate<String, User> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void sendUser(User user) {
        kafkaTemplate.send("users", user.getId(), user)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    logger.error("Failed to send user", ex);
                } else {
                    logger.info("User sent successfully");
                }
            });
    }
}

// Consumer service
@Service
public class UserConsumer {
    
    @KafkaListener(topics = "users", groupId = "user-consumer")
    public void consumeUser(User user) {
        logger.info("Received user: {}", user);
        processUser(user);
    }
}
```

### 3. **Type Mapping with JSON**

```java
public class JsonTypeMappingExamples {
    
    // Base event class
    @JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.PROPERTY,
        property = "type"
    )
    @JsonSubTypes({
        @JsonSubTypes.Type(value = UserCreatedEvent.class, name = "USER_CREATED"),
        @JsonSubTypes.Type(value = UserUpdatedEvent.class, name = "USER_UPDATED"),
        @JsonSubTypes.Type(value = UserDeletedEvent.class, name = "USER_DELETED")
    })
    public abstract class UserEvent {
        private String eventId;
        private Instant timestamp;
        
        // Getters, setters
    }
    
    // Concrete event types
    public class UserCreatedEvent extends UserEvent {
        private String userId;
        private String name;
        private String email;
        
        // Getters, setters
    }
    
    public class UserUpdatedEvent extends UserEvent {
        private String userId;
        private Map<String, Object> changes;
        
        // Getters, setters
    }
    
    public class UserDeletedEvent extends UserEvent {
        private String userId;
        private String reason;
        
        // Getters, setters
    }
    
    // Usage
    public void producePolymorphicEvents() {
        KafkaTemplate<String, UserEvent> template = kafkaTemplate();
        
        // Send different event types to same topic
        template.send("user-events", new UserCreatedEvent(/*...*/));
        template.send("user-events", new UserUpdatedEvent(/*...*/));
        template.send("user-events", new UserDeletedEvent(/*...*/));
    }
    
    @KafkaListener(topics = "user-events")
    public void consumePolymorphicEvents(UserEvent event) {
        // Type-based processing
        switch (event) {
            case UserCreatedEvent created -> handleUserCreated(created);
            case UserUpdatedEvent updated -> handleUserUpdated(updated);
            case UserDeletedEvent deleted -> handleUserDeleted(deleted);
            default -> logger.warn("Unknown event type: {}", event.getClass());
        }
    }
}
```

---

## Avro Serialization

### 1. **Avro Schema Definition**

```json
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {
      "name": "id",
      "type": "string"
    },
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "email",
      "type": "string"
    },
    {
      "name": "age",
      "type": "int"
    },
    {
      "name": "createdAt",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      }
    },
    {
      "name": "addresses",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "Address",
          "fields": [
            {
              "name": "street",
              "type": "string"
            },
            {
              "name": "city",
              "type": "string"
            },
            {
              "name": "zipCode",
              "type": "string"
            }
          ]
        }
      }
    },
    {
      "name": "metadata",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {}
    }
  ]
}
```

### 2. **Avro with Confluent Schema Registry**

```xml
<!-- Maven dependencies -->
<dependencies>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.5.0</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-schema-registry-client</artifactId>
        <version>7.5.0</version>
    </dependency>
</dependencies>
```

```java
public class AvroSerializationExample {
    
    // Producer configuration
    public Properties configureAvroProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            KafkaAvroSerializer.class.getName());
        
        // Schema Registry configuration
        props.put("schema.registry.url", "http://localhost:8081");
        
        return props;
    }
    
    // Producer
    public void produceAvroMessages() {
        try (KafkaProducer<String, User> producer = 
                new KafkaProducer<>(configureAvroProducer())) {
            
            // Create Avro object (generated from schema)
            User user = User.newBuilder()
                .setId("user-123")
                .setName("John Doe")
                .setEmail("john@example.com")
                .setAge(30)
                .setCreatedAt(Instant.now().toEpochMilli())
                .setAddresses(new ArrayList<>())
                .setMetadata(new HashMap<>())
                .build();
            
            ProducerRecord<String, User> record = new ProducerRecord<>(
                "users-avro",
                user.getId().toString(),
                user
            );
            
            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    logger.error("Failed to send Avro message", exception);
                } else {
                    logger.info("Sent Avro message to partition {} at offset {}", 
                        metadata.partition(), metadata.offset());
                }
            });
        }
    }
    
    // Consumer configuration
    public Properties configureAvroConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "avro-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            KafkaAvroDeserializer.class.getName());
        
        // Schema Registry configuration
        props.put("schema.registry.url", "http://localhost:8081");
        props.put("specific.avro.reader", "true");  // Use specific record
        
        return props;
    }
    
    // Consumer
    public void consumeAvroMessages() {
        try (KafkaConsumer<String, User> consumer = 
                new KafkaConsumer<>(configureAvroConsumer())) {
            
            consumer.subscribe(Collections.singletonList("users-avro"));
            
            while (true) {
                ConsumerRecords<String, User> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, User> record : records) {
                    User user = record.value();
                    logger.info("Received user: id={}, name={}, email={}", 
                        user.getId(), user.getName(), user.getEmail());
                    
                    processUser(user);
                }
            }
        }
    }
}
```

### 3. **Generic Avro Records**

```java
public class GenericAvroExample {
    
    // Using GenericRecord (no code generation)
    public void produceGenericAvro() throws IOException {
        // Load schema
        Schema schema = new Schema.Parser().parse(
            new File("src/main/resources/avro/user.avsc")
        );
        
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put("schema.registry.url", "http://localhost:8081");
        
        try (KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props)) {
            
            // Create generic record
            GenericRecord user = new GenericData.Record(schema);
            user.put("id", "user-123");
            user.put("name", "John Doe");
            user.put("email", "john@example.com");
            user.put("age", 30);
            user.put("createdAt", Instant.now().toEpochMilli());
            user.put("addresses", new ArrayList<>());
            user.put("metadata", new HashMap<>());
            
            ProducerRecord<String, GenericRecord> record = new ProducerRecord<>(
                "users-avro",
                user.get("id").toString(),
                user
            );
            
            producer.send(record);
        }
    }
    
    // Consume generic records
    public void consumeGenericAvro() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "generic-avro-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put("schema.registry.url", "http://localhost:8081");
        props.put("specific.avro.reader", "false");  // Use generic record
        
        try (KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("users-avro"));
            
            while (true) {
                ConsumerRecords<String, GenericRecord> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, GenericRecord> record : records) {
                    GenericRecord user = record.value();
                    
                    String id = user.get("id").toString();
                    String name = user.get("name").toString();
                    String email = user.get("email").toString();
                    
                    logger.info("Received user: id={}, name={}, email={}", 
                        id, name, email);
                }
            }
        }
    }
}
```

---

## Protobuf Serialization

### 1. **Protobuf Schema Definition**

```protobuf
// user.proto
syntax = "proto3";

package com.example.proto;

option java_package = "com.example.proto";
option java_outer_classname = "UserProto";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  google.protobuf.Timestamp created_at = 5;
  repeated Address addresses = 6;
  map<string, string> metadata = 7;
}

message Address {
  string street = 1;
  string city = 2;
  string zip_code = 3;
}
```

### 2. **Protobuf with Kafka**

```xml
<!-- Maven dependencies -->
<dependencies>
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.1</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-protobuf-serializer</artifactId>
        <version>7.5.0</version>
    </dependency>
</dependencies>

<!-- Protobuf Maven plugin -->
<build>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:3.25.1:exe:${os.detected.classifier}
                </protocArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```java
public class ProtobufSerializationExample {
    
    // Producer configuration
    public Properties configureProtobufProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaProtobufSerializer.class);
        props.put("schema.registry.url", "http://localhost:8081");
        
        return props;
    }
    
    // Producer
    public void produceProtobufMessages() {
        try (KafkaProducer<String, User> producer = 
                new KafkaProducer<>(configureProtobufProducer())) {
            
            // Create Protobuf message
            User user = User.newBuilder()
                .setId("user-123")
                .setName("John Doe")
                .setEmail("john@example.com")
                .setAge(30)
                .setCreatedAt(
                    Timestamp.newBuilder()
                        .setSeconds(Instant.now().getEpochSecond())
                        .build()
                )
                .build();
            
            ProducerRecord<String, User> record = new ProducerRecord<>(
                "users-protobuf",
                user.getId(),
                user
            );
            
            producer.send(record);
        }
    }
    
    // Consumer configuration
    public Properties configureProtobufConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "protobuf-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaProtobufDeserializer.class);
        props.put("schema.registry.url", "http://localhost:8081");
        props.put("specific.protobuf.value.type", User.class.getName());
        
        return props;
    }
    
    // Consumer
    public void consumeProtobufMessages() {
        try (KafkaConsumer<String, User> consumer = 
                new KafkaConsumer<>(configureProtobufConsumer())) {
            
            consumer.subscribe(Collections.singletonList("users-protobuf"));
            
            while (true) {
                ConsumerRecords<String, User> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, User> record : records) {
                    User user = record.value();
                    logger.info("Received user: id={}, name={}, email={}", 
                        user.getId(), user.getName(), user.getEmail());
                }
            }
        }
    }
}
```

---

## Schema Registry

### 1. **Schema Registry Integration**

```java
public class SchemaRegistryIntegration {
    
    /**
     * Schema Registry provides:
     * 1. Centralized schema storage
     * 2. Schema versioning
     * 3. Compatibility checking
     * 4. Schema evolution support
     */
    
    // Schema Registry client
    public class SchemaRegistryService {
        
        private final SchemaRegistryClient schemaRegistry;
        
        public SchemaRegistryService(String schemaRegistryUrl) {
            this.schemaRegistry = new CachedSchemaRegistryClient(
                schemaRegistryUrl,
                100  // Cache size
            );
        }
        
        // Register new schema
        public int registerSchema(String subject, Schema schema) throws Exception {
            return schemaRegistry.register(subject, schema);
        }
        
        // Get latest schema
        public Schema getLatestSchema(String subject) throws Exception {
            SchemaMetadata metadata = schemaRegistry.getLatestSchemaMetadata(subject);
            return new Schema.Parser().parse(metadata.getSchema());
        }
        
        // Get schema by version
        public Schema getSchemaByVersion(String subject, int version) throws Exception {
            SchemaMetadata metadata = schemaRegistry.getSchemaMetadata(subject, version);
            return new Schema.Parser().parse(metadata.getSchema());
        }
        
        // Check compatibility
        public boolean isCompatible(String subject, Schema newSchema) throws Exception {
            return schemaRegistry.testCompatibility(subject, newSchema);
        }
        
        // Update compatibility level
        public void updateCompatibility(String subject, String compatibility) throws Exception {
            schemaRegistry.updateCompatibility(subject, compatibility);
        }
    }
}
```

### 2. **Compatibility Modes**

```java
public class CompatibilityModes {
    
    /**
     * Compatibility Types:
     * 
     * BACKWARD (default):
     * - New schema can read data written by old schema
     * - Can delete fields (with defaults)
     * - Can add optional fields
     * 
     * FORWARD:
     * - Old schema can read data written by new schema
     * - Can add fields
     * - Can delete optional fields
     * 
     * FULL:
     * - Both BACKWARD and FORWARD
     * - Can only add/delete optional fields with defaults
     * 
     * BACKWARD_TRANSITIVE:
     * - New schema compatible with ALL previous versions
     * 
     * FORWARD_TRANSITIVE:
     * - Old schema compatible with ALL future versions
     * 
     * FULL_TRANSITIVE:
     * - Both BACKWARD_TRANSITIVE and FORWARD_TRANSITIVE
     * 
     * NONE:
     * - No compatibility checking
     */
    
    // Example: BACKWARD compatibility
    public void backwardCompatibilityExample() {
        /*
         * Original schema (v1):
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"}
         *   ]
         * }
         * 
         * New schema (v2) - BACKWARD compatible:
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"},
         *     {"name": "email", "type": "string", "default": ""}  // Added with default
         *   ]
         * }
         * 
         * ✅ New consumer can read old data (email will be "")
         */
    }
    
    // Example: FORWARD compatibility
    public void forwardCompatibilityExample() {
        /*
         * Original schema (v1):
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"},
         *     {"name": "age", "type": "int", "default": 0}
         *   ]
         * }
         * 
         * New schema (v2) - FORWARD compatible:
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"}
         *     // Removed optional field "age"
         *   ]
         * }
         * 
         * ✅ Old consumer can read new data (will use default for age)
         */
    }
}
```

### 3. **Subject Naming Strategies**

```java
public class SubjectNamingStrategies {
    
    /**
     * Subject Naming Strategies:
     * 
     * 1. TopicNameStrategy (default):
     *    - Subject: <topic-name>-value
     *    - Use: One schema per topic
     * 
     * 2. RecordNameStrategy:
     *    - Subject: <record-name>
     *    - Use: Multiple record types in same topic
     * 
     * 3. TopicRecordNameStrategy:
     *    - Subject: <topic-name>-<record-name>
     *    - Use: Multiple record types, topic-specific schemas
     */
    
    // Configuration
    public Properties configureNamingStrategy(String strategy) {
        Properties props = new Properties();
        props.put("schema.registry.url", "http://localhost:8081");
        props.put("value.subject.name.strategy", strategy);
        
        return props;
    }
    
    // Example: RecordNameStrategy for polymorphic events
    public void recordNameStrategyExample() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put("schema.registry.url", "http://localhost:8081");
        props.put("value.subject.name.strategy", 
            "io.confluent.kafka.serializers.subject.RecordNameStrategy");
        
        try (KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props)) {
            
            // Send different record types to same topic
            // Subject will be based on record name, not topic name
            
            producer.send(new ProducerRecord<>("events", userCreatedEvent));
            producer.send(new ProducerRecord<>("events", userUpdatedEvent));
            producer.send(new ProducerRecord<>("events", orderCreatedEvent));
        }
    }
}
```

---

## Custom Serializers

### 1. **Custom Serializer Implementation**

```java
public class CustomSerializerExample {
    
    // Custom efficient binary format
    public class UserBinarySerializer implements Serializer<User> {
        
        @Override
        public void configure(Map<String, ?> configs, boolean isKey) {
            // Configuration
        }
        
        @Override
        public byte[] serialize(String topic, User user) {
            if (user == null) {
                return null;
            }
            
            try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
                 DataOutputStream dos = new DataOutputStream(baos)) {
                
                // Write version byte (for future compatibility)
                dos.writeByte(1);
                
                // Write fields
                writeString(dos, user.getId());
                writeString(dos, user.getName());
                writeString(dos, user.getEmail());
                dos.writeInt(user.getAge());
                dos.writeLong(user.getCreatedAt().toEpochMilli());
                
                // Write addresses
                List<Address> addresses = user.getAddresses();
                dos.writeInt(addresses.size());
                for (Address address : addresses) {
                    writeString(dos, address.getStreet());
                    writeString(dos, address.getCity());
                    writeString(dos, address.getZipCode());
                }
                
                // Write metadata
                Map<String, String> metadata = user.getMetadata();
                dos.writeInt(metadata.size());
                for (Map.Entry<String, String> entry : metadata.entrySet()) {
                    writeString(dos, entry.getKey());
                    writeString(dos, entry.getValue());
                }
                
                return baos.toByteArray();
                
            } catch (IOException e) {
                throw new SerializationException("Error serializing User", e);
            }
        }
        
        private void writeString(DataOutputStream dos, String str) throws IOException {
            if (str == null) {
                dos.writeInt(-1);
            } else {
                byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
                dos.writeInt(bytes.length);
                dos.write(bytes);
            }
        }
        
        @Override
        public void close() {
            // Cleanup
        }
    }
    
    // Custom deserializer
    public class UserBinaryDeserializer implements Deserializer<User> {
        
        @Override
        public void configure(Map<String, ?> configs, boolean isKey) {
            // Configuration
        }
        
        @Override
        public User deserialize(String topic, byte[] data) {
            if (data == null) {
                return null;
            }
            
            try (ByteArrayInputStream bais = new ByteArrayInputStream(data);
                 DataInputStream dis = new DataInputStream(bais)) {
                
                // Read version
                byte version = dis.readByte();
                
                if (version == 1) {
                    return deserializeV1(dis);
                } else {
                    throw new SerializationException("Unknown version: " + version);
                }
                
            } catch (IOException e) {
                throw new SerializationException("Error deserializing User", e);
            }
        }
        
        private User deserializeV1(DataInputStream dis) throws IOException {
            String id = readString(dis);
            String name = readString(dis);
            String email = readString(dis);
            int age = dis.readInt();
            Instant createdAt = Instant.ofEpochMilli(dis.readLong());
            
            // Read addresses
            int addressCount = dis.readInt();
            List<Address> addresses = new ArrayList<>(addressCount);
            for (int i = 0; i < addressCount; i++) {
                String street = readString(dis);
                String city = readString(dis);
                String zipCode = readString(dis);
                addresses.add(new Address(street, city, zipCode));
            }
            
            // Read metadata
            int metadataCount = dis.readInt();
            Map<String, String> metadata = new HashMap<>(metadataCount);
            for (int i = 0; i < metadataCount; i++) {
                String key = readString(dis);
                String value = readString(dis);
                metadata.put(key, value);
            }
            
            return new User(id, name, email, age, createdAt, addresses, metadata);
        }
        
        private String readString(DataInputStream dis) throws IOException {
            int length = dis.readInt();
            if (length == -1) {
                return null;
            }
            byte[] bytes = new byte[length];
            dis.readFully(bytes);
            return new String(bytes, StandardCharsets.UTF_8);
        }
        
        @Override
        public void close() {
            // Cleanup
        }
    }
}
```

### 2. **Compression in Custom Serializer**

```java
public class CompressingSerializer<T> implements Serializer<T> {
    
    private final Serializer<T> innerSerializer;
    private final Compressor compressor;
    
    public CompressingSerializer(Serializer<T> innerSerializer) {
        this.innerSerializer = innerSerializer;
        this.compressor = new GZIPCompressor();
    }
    
    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        innerSerializer.configure(configs, isKey);
    }
    
    @Override
    public byte[] serialize(String topic, T data) {
        if (data == null) {
            return null;
        }
        
        // Serialize first
        byte[] serialized = innerSerializer.serialize(topic, data);
        
        // Then compress
        try {
            return compressor.compress(serialized);
        } catch (IOException e) {
            throw new SerializationException("Error compressing data", e);
        }
    }
    
    @Override
    public void close() {
        innerSerializer.close();
    }
    
    // Compressor interface
    interface Compressor {
        byte[] compress(byte[] data) throws IOException;
        byte[] decompress(byte[] data) throws IOException;
    }
    
    // GZIP implementation
    static class GZIPCompressor implements Compressor {
        
        @Override
        public byte[] compress(byte[] data) throws IOException {
            try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
                 GZIPOutputStream gzos = new GZIPOutputStream(baos)) {
                
                gzos.write(data);
                gzos.finish();
                return baos.toByteArray();
            }
        }
        
        @Override
        public byte[] decompress(byte[] data) throws IOException {
            try (ByteArrayInputStream bais = new ByteArrayInputStream(data);
                 GZIPInputStream gzis = new GZIPInputStream(bais);
                 ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
                
                byte[] buffer = new byte[1024];
                int len;
                while ((len = gzis.read(buffer)) > 0) {
                    baos.write(buffer, 0, len);
                }
                return baos.toByteArray();
            }
        }
    }
}

// Decompressing deserializer
public class DecompressingDeserializer<T> implements Deserializer<T> {
    
    private final Deserializer<T> innerDeserializer;
    private final CompressingSerializer.Compressor compressor;
    
    public DecompressingDeserializer(Deserializer<T> innerDeserializer) {
        this.innerDeserializer = innerDeserializer;
        this.compressor = new CompressingSerializer.GZIPCompressor();
    }
    
    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        innerDeserializer.configure(configs, isKey);
    }
    
    @Override
    public T deserialize(String topic, byte[] data) {
        if (data == null) {
            return null;
        }
        
        try {
            // Decompress first
            byte[] decompressed = compressor.decompress(data);
            
            // Then deserialize
            return innerDeserializer.deserialize(topic, decompressed);
            
        } catch (IOException e) {
            throw new SerializationException("Error decompressing data", e);
        }
    }
    
    @Override
    public void close() {
        innerDeserializer.close();
    }
}
```

---

## Error Handling

### 1. **Deserialization Error Handling**

```java
public class DeserializationErrorHandling {
    
    // ErrorHandlingDeserializer (Kafka 2.8+)
    public Properties configureErrorHandlingDeserializer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "error-handling-consumer");
        
        // Use ErrorHandlingDeserializer wrapper
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            ErrorHandlingDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            ErrorHandlingDeserializer.class);
        
        // Configure actual deserializers
        props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, 
            StringDeserializer.class);
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, 
            JsonDeserializer.class);
        
        return props;
    }
    
    // Consume with error handling
    @KafkaListener(topics = "users")
    public void consumeWithErrorHandling(
            ConsumerRecord<String, User> record,
            @Header(KafkaHeaders.RECEIVED_KEY) byte[] keyBytes,
            @Header(KafkaHeaders.RECEIVED_VALUE) byte[] valueBytes) {
        
        // Check for deserialization errors
        if (record.value() == null) {
            DeserializationException exception = 
                (DeserializationException) record.headers()
                    .lastHeader("value.deserializer.exception").value();
            
            if (exception != null) {
                logger.error("Deserialization failed for key: {}", 
                    record.key(), exception);
                
                // Send to DLQ
                sendToDeadLetterQueue(record.topic(), keyBytes, valueBytes, exception);
                return;
            }
        }
        
        // Normal processing
        processUser(record.value());
    }
    
    // Dead Letter Queue
    public void sendToDeadLetterQueue(
            String originalTopic,
            byte[] key,
            byte[] value,
            Exception exception) {
        
        ProducerRecord<byte[], byte[]> dlqRecord = new ProducerRecord<>(
            originalTopic + ".DLQ",
            key,
            value
        );
        
        // Add error metadata
        dlqRecord.headers().add("original-topic", originalTopic.getBytes());
        dlqRecord.headers().add("exception-message", exception.getMessage().getBytes());
        dlqRecord.headers().add("exception-stacktrace", 
            getStackTraceAsString(exception).getBytes());
        dlqRecord.headers().add("timestamp", 
            String.valueOf(System.currentTimeMillis()).getBytes());
        
        dlqProducer.send(dlqRecord);
    }
}
```

### 2. **Custom Error Handler**

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    // Consumer error handler
    @Bean
    public DefaultErrorHandler errorHandler(
            KafkaTemplate<String, Object> kafkaTemplate) {
        
        // Create dead letter publisher
        DeadLetterPublishingRecoverer recoverer = 
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, exception) -> {
                    // Determine DLQ topic based on exception type
                    if (exception instanceof DeserializationException) {
                        return new TopicPartition(
                            record.topic() + ".deserialization.DLQ", 
                            record.partition()
                        );
                    } else if (exception instanceof RecoverableDataAccessException) {
                        return new TopicPartition(
                            record.topic() + ".retry.DLQ",
                            record.partition()
                        );
                    } else {
                        return new TopicPartition(
                            record.topic() + ".DLQ",
                            record.partition()
                        );
                    }
                });
        
        // Configure backoff
        FixedBackOff backOff = new FixedBackOff(1000L, 3L);  // 1 second, 3 attempts
        
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);
        
        // Don't retry deserialization errors
        errorHandler.addNotRetryableExceptions(DeserializationException.class);
        
        return errorHandler;
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, User> kafkaListenerContainerFactory(
            ConsumerFactory<String, User> consumerFactory,
            DefaultErrorHandler errorHandler) {
        
        ConcurrentKafkaListenerContainerFactory<String, User> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler);
        
        return factory;
    }
}
```

---

## Schema Evolution

### 1. **Avro Schema Evolution**

```java
public class AvroSchemaEvolution {
    
    /**
     * Schema Evolution Rules:
     * 
     * BACKWARD Compatible (default):
     * ✅ Delete fields (consumer must have default)
     * ✅ Add optional fields (with default values)
     * ❌ Cannot add required fields
     * ❌ Cannot delete fields without defaults
     * 
     * FORWARD Compatible:
     * ✅ Add fields
     * ✅ Delete optional fields
     * ❌ Cannot delete required fields
     * 
     * FULL Compatible:
     * ✅ Add optional fields with defaults
     * ✅ Delete optional fields with defaults
     */
    
    // Example: Adding optional field (BACKWARD compatible)
    public void addOptionalField() {
        /*
         * V1:
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"}
         *   ]
         * }
         * 
         * V2 (BACKWARD compatible):
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"},
         *     {"name": "email", "type": "string", "default": ""}
         *   ]
         * }
         * 
         * ✅ New consumer can read old data (email = "")
         */
    }
    
    // Example: Using union types for nullable fields
    public void nullableFields() {
        /*
         * V1:
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"}
         *   ]
         * }
         * 
         * V2 (BACKWARD compatible):
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"},
         *     {"name": "email", "type": ["null", "string"], "default": null}
         *   ]
         * }
         * 
         * Union type ["null", "string"] allows null values
         * Default is null, so old messages work fine
         */
    }
    
    // Example: Evolving field types
    public void evolvingFieldTypes() {
        /*
         * Type promotion (FULL compatible):
         * - int → long
         * - int → float
         * - int → double
         * - long → float
         * - long → double
         * - float → double
         * - string → bytes
         * 
         * V1:
         * {"name": "age", "type": "int"}
         * 
         * V2 (FULL compatible):
         * {"name": "age", "type": "long"}
         * 
         * ✅ Works because int can be promoted to long
         */
    }
}
```

### 2. **Versioning Strategies**

```java
public class VersioningStrategies {
    
    // Strategy 1: Embedded version field
    public class EmbeddedVersioning {
        
        /*
         * Schema:
         * {
         *   "type": "record",
         *   "name": "User",
         *   "fields": [
         *     {"name": "schemaVersion", "type": "int", "default": 1},
         *     {"name": "id", "type": "string"},
         *     {"name": "name", "type": "string"}
         *   ]
         * }
         */
        
        public User deserialize(byte[] data) {
            GenericRecord record = deserializeToGenericRecord(data);
            int version = (int) record.get("schemaVersion");
            
            return switch (version) {
                case 1 -> deserializeV1(record);
                case 2 -> deserializeV2(record);
                default -> throw new IllegalArgumentException("Unknown version: " + version);
            };
        }
    }
    
    // Strategy 2: Multiple topics per version
    public class TopicVersioning {
        
        // v1 → users-v1
        // v2 → users-v2
        
        public void produceV1() {
            producer.send(new ProducerRecord<>("users-v1", userV1));
        }
        
        public void produceV2() {
            producer.send(new ProducerRecord<>("users-v2", userV2));
        }
    }
    
    // Strategy 3: Schema Registry versioning (recommended)
    public class SchemaRegistryVersioning {
        
        /*
         * Schema Registry automatically:
         * - Assigns version numbers
         * - Validates compatibility
         * - Stores all versions
         * - Allows consumers to read any version
         * 
         * Producer: Writes with latest schema
         * Consumer: Reads with compatible schema (can be older)
         */
    }
}
```

---

## Performance Optimization

### 1. **Batch Serialization**

```java
public class BatchSerialization {
    
    // Kafka batches records automatically, but you can optimize further
    
    public void configureForHighThroughput() {
        Properties props = new Properties();
        
        // Increase batch size (default: 16KB)
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);  // 32KB
        
        // Wait up to 10ms for batch to fill
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        
        // Compression (reduces network I/O)
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        // Options: none, gzip, snappy, lz4, zstd
        
        // Buffer memory for batching
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);  // 64MB
    }
}
```

### 2. **Serialization Caching**

```java
public class CachingSerializer<T> implements Serializer<T> {
    
    private final Serializer<T> delegate;
    private final Cache<T, byte[]> cache;
    
    public CachingSerializer(Serializer<T> delegate) {
        this.delegate = delegate;
        this.cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .build();
    }
    
    @Override
    public byte[] serialize(String topic, T data) {
        if (data == null) {
            return null;
        }
        
        // Check cache first
        byte[] cached = cache.getIfPresent(data);
        if (cached != null) {
            return cached;
        }
        
        // Serialize and cache
        byte[] serialized = delegate.serialize(topic, data);
        cache.put(data, serialized);
        
        return serialized;
    }
    
    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        delegate.configure(configs, isKey);
    }
    
    @Override
    public void close() {
        delegate.close();
        cache.invalidateAll();
    }
}
```

### 3. **Compression Comparison**

```
Format Size Comparison (1000 user records):

Uncompressed:
├─ JSON:      ~250 KB
├─ Avro:      ~100 KB
└─ Protobuf:  ~80 KB

With Snappy compression:
├─ JSON:      ~60 KB  (76% reduction)
├─ Avro:      ~45 KB  (55% reduction)
└─ Protobuf:  ~40 KB  (50% reduction)

With GZIP compression:
├─ JSON:      ~40 KB  (84% reduction)
├─ Avro:      ~35 KB  (65% reduction)
└─ Protobuf:  ~30 KB  (62% reduction)

Compression Speed:
├─ Snappy:    Fastest (compression + decompression)
├─ LZ4:       Very fast
├─ Zstd:      Fast with good ratio
└─ GZIP:      Slowest but best compression

Recommendation: Snappy for most cases (good balance)
```

---

## Testing Strategies

### 1. **Unit Testing Serializers**

```java
public class SerializerTest {
    
    private JsonSerializer<User> serializer;
    private JsonDeserializer<User> deserializer;
    
    @BeforeEach
    void setUp() {
        serializer = new JsonSerializer<>();
        deserializer = new JsonDeserializer<>(User.class);
    }
    
    @Test
    void shouldSerializeAndDeserialize() {
        // Arrange
        User user = new User("123", "John Doe", "john@example.com");
        
        // Act
        byte[] serialized = serializer.serialize("users", user);
        User deserialized = deserializer.deserialize("users", serialized);
        
        // Assert
        assertNotNull(serialized);
        assertEquals(user.getId(), deserialized.getId());
        assertEquals(user.getName(), deserialized.getName());
        assertEquals(user.getEmail(), deserialized.getEmail());
    }
    
    @Test
    void shouldHandleNullValue() {
        byte[] serialized = serializer.serialize("users", null);
        assertNull(serialized);
        
        User deserialized = deserializer.deserialize("users", null);
        assertNull(deserialized);
    }
    
    @Test
    void shouldHandleSpecialCharacters() {
        User user = new User("123", "John \"The Boss\" O'Brien", "john@example.com");
        
        byte[] serialized = serializer.serialize("users", user);
        User deserialized = deserializer.deserialize("users", serialized);
        
        assertEquals(user.getName(), deserialized.getName());
    }
    
    @Test
    void shouldThrowOnInvalidData() {
        byte[] invalidData = "{invalid json}".getBytes();
        
        assertThrows(SerializationException.class, () -> 
            deserializer.deserialize("users", invalidData)
        );
    }
}
```

### 2. **Integration Testing with Embedded Kafka**

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"users"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "port=9092"
    }
)
public class KafkaSerializationIntegrationTest {
    
    @Autowired
    private KafkaTemplate<String, User> kafkaTemplate;
    
    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;
    
    private KafkaConsumer<String, User> consumer;
    
    @BeforeEach
    void setUp() {
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka
        );
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class);
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
            JsonDeserializer.class);
        consumerProps.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        consumerProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, User.class.getName());
        
        consumer = new KafkaConsumer<>(consumerProps);
        embeddedKafka.consumeFromAnEmbeddedTopic(consumer, "users");
    }
    
    @AfterEach
    void tearDown() {
        consumer.close();
    }
    
    @Test
    void shouldSerializeAndDeserializeThroughKafka() {
        // Arrange
        User user = new User("123", "John Doe", "john@example.com");
        
        // Act
        kafkaTemplate.send("users", user.getId(), user);
        
        // Assert
        ConsumerRecord<String, User> record = KafkaTestUtils.getSingleRecord(
            consumer, "users", Duration.ofSeconds(10)
        );
        
        assertEquals(user.getId(), record.key());
        assertEquals(user.getName(), record.value().getName());
        assertEquals(user.getEmail(), record.value().getEmail());
    }
}
```

### 3. **Schema Evolution Testing**

```java
public class SchemaEvolutionTest {
    
    @Test
    void shouldReadOldDataWithNewSchema() throws Exception {
        // V1 schema
        Schema schemaV1 = new Schema.Parser().parse("""
            {
              "type": "record",
              "name": "User",
              "fields": [
                {"name": "id", "type": "string"},
                {"name": "name", "type": "string"}
              ]
            }
            """);
        
        // V2 schema (added email with default)
        Schema schemaV2 = new Schema.Parser().parse("""
            {
              "type": "record",
              "name": "User",
              "fields": [
                {"name": "id", "type": "string"},
                {"name": "name", "type": "string"},
                {"name": "email", "type": "string", "default": ""}
              ]
            }
            """);
        
        // Create record with V1 schema
        GenericRecord userV1 = new GenericData.Record(schemaV1);
        userV1.put("id", "123");
        userV1.put("name", "John Doe");
        
        // Serialize with V1
        byte[] bytes = serializeAvro(userV1, schemaV1);
        
        // Deserialize with V2
        GenericRecord userV2 = deserializeAvro(bytes, schemaV1, schemaV2);
        
        // Assert
        assertEquals("123", userV2.get("id").toString());
        assertEquals("John Doe", userV2.get("name").toString());
        assertEquals("", userV2.get("email").toString());  // Default value
    }
    
    private byte[] serializeAvro(GenericRecord record, Schema schema) throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DatumWriter<GenericRecord> writer = new GenericDatumWriter<>(schema);
        Encoder encoder = EncoderFactory.get().binaryEncoder(baos, null);
        writer.write(record, encoder);
        encoder.flush();
        return baos.toByteArray();
    }
    
    private GenericRecord deserializeAvro(byte[] bytes, Schema writerSchema, Schema readerSchema) 
            throws Exception {
        DatumReader<GenericRecord> reader = new GenericDatumReader<>(writerSchema, readerSchema);
        Decoder decoder = DecoderFactory.get().binaryDecoder(bytes, null);
        return reader.read(null, decoder);
    }
}
```

---

## Best Practices

### ✅ DO

1. **Use Schema Registry for production**
   - Centralized schema management
   - Automatic compatibility checking
   - Version control

2. **Choose appropriate format**
   - Avro: Best for schema evolution
   - Protobuf: Smallest size, fastest
   - JSON: Debugging, human-readable

3. **Enable compression**
   - Snappy for balance
   - GZIP for maximum compression
   - LZ4 for speed

4. **Handle deserialization errors**
   - Use ErrorHandlingDeserializer
   - Implement Dead Letter Queue
   - Log failures with context

5. **Version your schemas**
   - Add optional fields with defaults
   - Use schema evolution rules
   - Test compatibility

6. **Configure batching**
   - batch.size for throughput
   - linger.ms for latency/throughput tradeoff
   - compression.type for efficiency

7. **Test serialization**
   - Unit test serializers
   - Integration test with Kafka
   - Test schema evolution

8. **Monitor performance**
   - Serialization time
   - Message size
   - Deserialization errors

### ❌ DON'T

1. **Don't use Java serialization**
   - Not cross-platform
   - Security issues
   - Poor performance

2. **Don't skip Schema Registry**
   - Manual schema management is error-prone
   - Lose compatibility checking
   - Hard to evolve schemas

3. **Don't ignore backward compatibility**
   - Breaking changes break consumers
   - Deploy schema changes carefully
   - Test with old/new versions

4. **Don't serialize large objects**
   - Split into multiple messages
   - Use external storage for large data
   - Reference by ID

5. **Don't swallow deserialization errors**
   - Log errors with context
   - Send to DLQ
   - Alert on high error rates

6. **Don't hardcode schemas**
   - Use Schema Registry
   - Generate code from schemas
   - Version control schema files

7. **Don't forget nullability**
   - Use union types in Avro
   - Set defaults appropriately
   - Handle null values

8. **Don't mix serialization formats**
   - One format per topic
   - Document format choice
   - Consider migration path

---

## Decision Tree

```
Serialization Format Selection:

START: What are your requirements?

├─ Need human readability for debugging?
│  └─► Use JSON
│      - Easy to debug
│      - Widely supported
│      - Larger size
│      - Slower performance
│
├─ Need cross-language compatibility?
│  │
│  ├─ Need smallest size / fastest?
│  │  └─► Use Protobuf
│  │      - Smallest size
│  │      - Fastest performance
│  │      - Good schema evolution
│  │      - Requires schema definition
│  │
│  └─ Need best schema evolution?
│     └─► Use Avro
│         - Excellent schema evolution
│         - Compact binary format
│         - Schema Registry integration
│         - Good performance
│
├─ Simple use case (strings, numbers)?
│  └─► Use Built-in Serializers
│      - StringSerializer
│      - IntegerSerializer
│      - LongSerializer
│
├─ Legacy Java-only system?
│  └─► Use JSON (not Java Serialization!)
│      - Java Serialization has issues
│      - JSON more maintainable
│
└─ Special requirements?
   └─► Custom Serializer
       - Specific binary format
       - Integration with existing system
       - Encryption/compression needs
```

### Quick Reference Matrix

| Requirement | Recommended Format | Alternative |
|-------------|-------------------|-------------|
| Schema evolution | Avro + Schema Registry | Protobuf |
| Smallest size | Protobuf | Avro |
| Fastest performance | Protobuf | Avro |
| Human readable | JSON | - |
| Debugging | JSON | - |
| Cross-language | Avro, Protobuf | JSON |
| Simplest setup | JSON | Built-in |
| Production system | Avro + Schema Registry | Protobuf |
| Legacy integration | Custom Serializer | JSON |

---

## Conclusion

**Key Takeaways:**

1. **For Production**: Use **Avro with Schema Registry**
   - Best schema evolution
   - Compatibility checking
   - Good performance

2. **For High Performance**: Use **Protobuf**
   - Smallest size
   - Fastest serialization
   - When schema evolution less critical

3. **For Debugging/Development**: Use **JSON**
   - Easy to read
   - Simple to debug
   - Good for prototyping

4. **Always**:
   - Enable compression (Snappy recommended)
   - Handle deserialization errors
   - Test schema evolution
   - Monitor performance metrics

**Remember**: Serialization format is a critical decision that impacts:
- Performance (throughput, latency)
- Storage costs
- Operational complexity
- Schema evolution capability
- Cross-platform compatibility

Choose based on your specific needs, but default to **Avro + Schema Registry** for most production systems.
