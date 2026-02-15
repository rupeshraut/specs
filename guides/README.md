# 📚 Comprehensive Technical Guides

In-depth, tutorial-style guides for mastering specific technologies, patterns, and practices in enterprise Java development. While cheatsheets provide quick reference material, these guides offer comprehensive learning paths with detailed explanations, examples, and best practices.

---

## 📖 Table of Contents

### ☕ Java Language & Core APIs

- [**Modern Java Concurrency**](modern-java-concurrency-guide.md)
  Comprehensive guide to concurrency in modern Java (Java 8 through Java 21+). Covers thread safety, virtual threads, structured concurrency, and async patterns.

- [**CompletableFuture Mastery**](completable-future-guide.md)
  Deep dive into asynchronous programming with CompletableFuture. Creating, chaining, combining, error handling, and advanced patterns.

- [**Modern Java Date/Time API**](modern-java-datetime-guide.md)
  Complete guide to the java.time API (Java 8+). LocalDate, LocalDateTime, ZonedDateTime, Duration, Period, and timezone handling.

- [**Exception Design and Handling**](exception-design-guide.md)
  Designing robust exception hierarchies and handling errors effectively. Custom exceptions, exception translation, and error recovery patterns.

### 🧩 Design & Architecture

- [**Design Patterns Decision Tree**](design-patterns-decision-tree.md)
  Interactive guide to choosing the right design pattern for your scenario. Covers creational, structural, and behavioral patterns with decision flows.

### 🧪 Testing

- [**Modern Java Unit Testing**](modern-java-testing-guide.md)
  Mastering JUnit 5, Mockito, AssertJ, and Testcontainers. Test structure, mocking strategies, assertions, and integration testing patterns.

### 📊 Data Storage & Caching

- [**Modern MongoDB Query Patterns**](mongodb-query-guide.md)
  Complete guide to MongoDB queries, aggregation framework, indexing, and performance optimization. CRUD operations, complex aggregations, and schema design.

- [**Modern Redis with Lettuce**](redis-lettuce-guide.md)
  Mastering Redis with Lettuce client and Spring Data Redis. Caching patterns, data structures, pub/sub, distributed locks, and performance tuning.

### 🔥 Messaging & Events

- [**Kafka Serialization & Deserialization**](kafka-serialization-guide.md)
  Complete guide to Kafka serialization, schema management, and data evolution. Avro, JSON, Protobuf, Schema Registry, and compatibility strategies.

### 🛡️ Resilience & Reliability

- [**Modern Resilience with Resilience4j**](resilience4j-complete-guide.md)
  Comprehensive guide to building resilient systems with Resilience4j. Circuit breakers, retries, rate limiters, bulkheads, and time limiters with real-world examples.

- [**Resilience4j (Alternative Guide)**](resilience4j-guide.md)
  Additional Resilience4j guide with Kafka, database, and REST API integration patterns.

- [**Resilience4j - Programmatic Usage**](resilience4j-programmatic-guide.md)
  Using Resilience4j without Spring framework. Standalone setup and programmatic configuration for non-Spring applications.

### 🛠️ Development Tools & Practices

- [**Modern Git Usage**](git-modern-guide.md)
  Mastering Git workflows, best practices, advanced techniques, and team collaboration strategies. Branching, merging, rebasing, and conflict resolution.

- [**Java Application Profiling**](java-profiling-guide.md)
  Complete guide to profiling Java applications. JFR (Java Flight Recorder), Async Profiler, GC log analysis, and performance investigation techniques.

---

## 🎯 How to Use These Guides

### Guides vs Cheatsheets

| Aspect | Guides (this folder) | Cheatsheets (parent folder) |
|--------|---------------------|----------------------------|
| **Purpose** | Learn deeply | Quick reference |
| **Length** | Comprehensive (20-100+ sections) | Concise (5-15 patterns) |
| **Use When** | Learning new technology | Already familiar, need reminder |
| **Format** | Tutorial with examples | Decision frameworks + snippets |
| **Reading Time** | 30-60+ minutes | 5-15 minutes |

### Learning Paths

**Path 1: Java Fundamentals Mastery**

1. [Modern Java Date/Time API](modern-java-datetime-guide.md)
2. [Exception Design and Handling](exception-design-guide.md)
3. [Modern Java Concurrency](modern-java-concurrency-guide.md)
4. [CompletableFuture Mastery](completable-future-guide.md)

**Path 2: Distributed Systems Developer**

1. [Kafka Serialization & Deserialization](kafka-serialization-guide.md)
2. [Modern Resilience with Resilience4j](resilience4j-complete-guide.md)
3. [Modern MongoDB Query Patterns](mongodb-query-guide.md)
4. [Modern Redis with Lettuce](redis-lettuce-guide.md)

**Path 3: Quality & Best Practices**

1. [Design Patterns Decision Tree](design-patterns-decision-tree.md)
2. [Exception Design and Handling](exception-design-guide.md)
3. [Modern Java Unit Testing](modern-java-testing-guide.md)
4. [Modern Git Usage](git-modern-guide.md)

**Path 4: Performance Engineering**

1. [Java Application Profiling](java-profiling-guide.md)
2. [Modern Java Concurrency](modern-java-concurrency-guide.md)
3. [Modern MongoDB Query Patterns](mongodb-query-guide.md)
4. [Modern Redis with Lettuce](redis-lettuce-guide.md)

---

## 🚀 Quick Start Recommendations

### I want to learn...

- **Asynchronous programming** → Start with [CompletableFuture Mastery](completable-future-guide.md)
- **Virtual threads & modern concurrency** → Start with [Modern Java Concurrency](modern-java-concurrency-guide.md)
- **Design patterns** → Start with [Design Patterns Decision Tree](design-patterns-decision-tree.md)
- **Testing best practices** → Start with [Modern Java Unit Testing](modern-java-testing-guide.md)
- **Kafka data handling** → Start with [Kafka Serialization](kafka-serialization-guide.md)
- **Building resilient systems** → Start with [Resilience4j Complete Guide](resilience4j-complete-guide.md)
- **MongoDB optimization** → Start with [MongoDB Query Patterns](mongodb-query-guide.md)
- **Redis caching** → Start with [Redis with Lettuce](redis-lettuce-guide.md)
- **Git mastery** → Start with [Modern Git Usage](git-modern-guide.md)
- **Performance troubleshooting** → Start with [Java Profiling](java-profiling-guide.md)

---

## 📝 Guide Structure

Each guide follows a consistent format:

1. **Fundamentals** — Core concepts and mental models
2. **Basic Usage** — Getting started with simple examples
3. **Intermediate Patterns** — Common real-world scenarios
4. **Advanced Techniques** — Complex use cases and optimizations
5. **Best Practices** — Production-proven patterns and anti-patterns
6. **Troubleshooting** — Common issues and solutions
7. **Performance** — Optimization techniques
8. **Real-World Examples** — Complete working code samples

---

## 🔗 Related Resources

- **Parent Folder** — [Quick-reference cheatsheets](../README.md) for fast lookups
- **Stack Context** — All guides assume Java 21+, Spring Boot 3.x, and modern tooling

---

## 📄 License

Internal reference documentation for enterprise Java development.
