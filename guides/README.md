# 📚 Comprehensive Technical Guides

In-depth, tutorial-style guides for mastering specific technologies, patterns, and practices in enterprise Java development. While cheatsheets provide quick reference material, these guides offer comprehensive learning paths with detailed explanations, examples, and best practices.

---

## 📖 Table of Contents

### ☕ Java Language & Core APIs

- [**Effective Java Generics**](java-generics-guide.md)
  Master Java generics from fundamentals to advanced patterns. Type parameters, bounds, wildcards, PECS principle, type erasure, generics with records and sealed types, and modern Java features.

- [**Java Optional Mastery**](java-optional-guide.md)
  Complete guide to using Optional effectively. Creating, transforming, chaining, best practices, and avoiding common pitfalls with null-safe programming.

- [**Java Functional Interfaces**](java-functional-interfaces-guide.md)
  Comprehensive guide to functional interfaces and lambda expressions. Predicate, Function, Consumer, Supplier, custom functional interfaces, and method references.

- [**Java Enums Deep Dive**](java-enum-guide.md)
  Master Java enums from basics to advanced patterns. Enum methods, EnumSet, EnumMap, strategy pattern with enums, and state machines.

- [**Modern Java Concurrency**](modern-java-concurrency-guide.md)
  Comprehensive guide to concurrency in modern Java (Java 8 through Java 21+). Covers thread safety, virtual threads, structured concurrency, and async patterns.

- [**CompletableFuture Mastery**](completable-future-guide.md)
  Deep dive into asynchronous programming with CompletableFuture. Creating, chaining, combining, error handling, and advanced patterns.

- [**Java Concurrency Utilities**](java-concurrency-utilities-guide.md)
  Master Java concurrency utilities including synchronizers (CountDownLatch, CyclicBarrier, Semaphore, Phaser), locks (ReentrantLock, ReadWriteLock, StampedLock), concurrent collections, and atomic variables.

- [**Modern Java Date/Time API**](modern-java-datetime-guide.md)
  Complete guide to the java.time API (Java 8+). LocalDate, LocalDateTime, ZonedDateTime, Duration, Period, and timezone handling.

- [**Exception Design and Handling**](exception-design-guide.md)
  Designing robust exception hierarchies and handling errors effectively. Custom exceptions, exception translation, and error recovery patterns.

### 🧩 Design & Architecture

- [**Clean & Hexagonal Architecture Deep Dive**](clean-architecture-guide.md)
  Comprehensive guide to implementing Clean and Hexagonal Architecture. Domain layer, ports & adapters, dependency rules, package structure, and migration patterns.

- [**Domain-Driven Design (DDD) Complete Guide**](domain-driven-design-guide.md)
  Strategic and tactical DDD patterns. Bounded contexts, aggregates, entities, value objects, domain events, and ubiquitous language.

- [**Design Patterns Decision Tree**](design-patterns-decision-tree.md)
  Interactive guide to choosing the right design pattern for your scenario. Covers creational, structural, and behavioral patterns with decision flows.

- [**Event-Driven Architecture Mastery**](event-driven-architecture-guide.md)
  Comprehensive guide to designing event-driven systems. Sagas, event sourcing, CQRS, outbox pattern, idempotency, and distributed patterns.

- [**Distributed Transactions & Consistency**](distributed-transactions-consistency-guide.md)
  Complete guide to distributed transactions, saga pattern, eventual consistency, CAP theorem, outbox pattern, idempotency, and compensating transactions.

### 🌐 API Design & Integration

- [**REST API Design & Implementation**](rest-api-design-guide.md)
  Production-grade REST API design patterns. Resource modeling, HTTP methods, status codes, pagination, versioning, HATEOAS, OpenAPI documentation, and best practices.

- [**API Gateway & Service Mesh Patterns**](api-gateway-service-mesh-guide.md)
  Comprehensive guide to API gateways and service mesh. Spring Cloud Gateway, Istio patterns, traffic management, mTLS security, rate limiting, and circuit breaking.

### 🌱 Spring Framework

- [**Spring Security Complete Guide**](spring-security-guide.md)
  Production-grade Spring Security with OAuth2, JWT, method security, and authentication patterns. CORS, CSRF, session management, and security best practices.

- [**Spring WebFlux & Reactive Programming**](spring-webflux-reactive-guide.md)
  Comprehensive guide to reactive programming with Spring WebFlux and Project Reactor. Mono, Flux, reactive WebClient, backpressure, and performance patterns.

- [**Project Reactor**](project-reactor-guide.md)
  Master reactive programming with Project Reactor. Mono, Flux, transformation and filtering operators, backpressure, schedulers, error handling, and testing reactive code.

- [**Spring Boot Production Mastery**](spring-boot-production-guide.md)
  Production-ready Spring Boot configuration, application lifecycle, actuator deep dive, graceful shutdown, startup optimization, and operational patterns.

### 🧪 Testing

- [**Modern Java Unit Testing**](modern-java-testing-guide.md)
  Mastering JUnit 5, Mockito, AssertJ, and Testcontainers. Test structure, mocking strategies, assertions, and integration testing patterns.

### 📊 Data Storage & Caching

- [**Modern MongoDB Query Patterns**](mongodb-query-guide.md)
  Complete guide to MongoDB queries, aggregation framework, indexing, and performance optimization. CRUD operations, complex aggregations, and schema design.

- [**Modern Redis with Lettuce**](redis-lettuce-guide.md)
  Mastering Redis with Lettuce client and Spring Data Redis. Caching patterns, data structures, pub/sub, distributed locks, and performance tuning.

- [**Database Migration & Schema Evolution**](database-migration-guide.md)
  Complete guide to Flyway and Liquibase. Zero-downtime migrations, rollback strategies, testing migrations, and production deployment patterns.

### 🔥 Messaging & Events

- [**Apache Kafka Deep Dive**](kafka-deep-dive-guide.md)
  Comprehensive guide to Kafka internals, architecture, producer/consumer configuration, exactly-once semantics, Kafka Streams, performance tuning, and operations.

- [**Kafka Serialization & Deserialization**](kafka-serialization-guide.md)
  Complete guide to Kafka serialization, schema management, and data evolution. Avro, JSON, Protobuf, Schema Registry, and compatibility strategies.

- [**Kafka Backpressure Handling**](kafka-backpressure-guide.md)
  Managing backpressure in Kafka consumers, preventing lag, pause/resume patterns, rate limiting, parallel processing, and Reactor Kafka integration.

- [**Kafka Zero Message Loss**](kafka-zero-loss-guide.md)
  Implementing zero message loss with producer/consumer configuration, idempotence, transactions, error handling, retry patterns, DLQ, and exactly-once semantics.

- [**Apache Flink 2.0**](apache-flink-guide.md)
  Master stream and batch processing with Apache Flink 2.0. DataStream API, windowing, state management, checkpointing, watermarks, Table API/SQL, connectors, exactly-once semantics, and performance optimization.

- [**Flink Checkpointing and Operators**](flink-checkpointing-guide.md)
  Deep dive into Flink checkpointing configuration, state backends, aligned vs unaligned checkpoints, synchronous vs asynchronous operators, barrier alignment, incremental checkpoints, savepoints, and performance optimization.

### 🔧 Infrastructure & Operations

- [**Docker & Kubernetes for Java Services**](docker-kubernetes-guide.md)
  Comprehensive guide to containerizing and orchestrating Java applications. Multi-stage builds, health probes, resource management, deployment strategies, and production patterns.

- [**Observability Engineering**](observability-engineering-guide.md)
  Production-grade observability with metrics, logging, and tracing. Micrometer, distributed tracing, structured logging, SLI/SLO/SLA, alerting strategies, and dashboards.

### 🛡️ Resilience & Reliability

- [**Modern Resilience with Resilience4j**](resilience4j-complete-guide.md)
  Comprehensive guide to building resilient systems with Resilience4j. Circuit breakers, retries, rate limiters, bulkheads, and time limiters with real-world examples.

- [**Resilience4j (Alternative Guide)**](resilience4j-guide.md)
  Additional Resilience4j guide with Kafka, database, and REST API integration patterns.

- [**Resilience4j - Programmatic Usage**](resilience4j-programmatic-guide.md)
  Using Resilience4j without Spring framework. Standalone setup and programmatic configuration for non-Spring applications.

- [**Chaos Engineering for Java**](chaos-engineering-guide.md)
  Production chaos engineering practices with Chaos Monkey, Chaos Toolkit, Litmus Chaos. Simulating failures, testing resilience patterns, game days, and safe production chaos.

### 🔒 Security

- [**Application Security Guide**](application-security-guide.md)
  Comprehensive security guide covering OWASP Top 10, injection prevention, authentication, authorization, cryptography, dependency security, and production hardening.

- [**Secrets Management & Configuration**](secrets-management-guide.md)
  Complete guide to managing secrets with HashiCorp Vault, AWS Secrets Manager, Spring Cloud Config, Kubernetes Secrets, secret rotation, and encryption patterns.

### 🛠️ Development Tools & Practices

- [**Modern Git Usage**](git-modern-guide.md)
  Mastering Git workflows, best practices, advanced techniques, and team collaboration strategies. Branching, merging, rebasing, and conflict resolution.

- [**Java Application Profiling**](java-profiling-guide.md)
  Complete guide to profiling Java applications. JFR (Java Flight Recorder), Async Profiler, GC log analysis, and performance investigation techniques.

### ⚡ Performance & Optimization

- [**JVM Performance Engineering**](jvm-performance-engineering-guide.md)
  Comprehensive guide to JVM performance tuning. GC algorithms (G1GC, ZGC, Shenandoah), memory management, heap analysis, thread profiling, container-aware JVM, and production optimization.

- [**Performance Testing Mastery**](performance-testing-guide.md)
  Complete guide to performance testing with JMeter, Gatling, and K6. Load testing, stress testing, spike testing, soak testing, metrics analysis, and CI/CD integration patterns.

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

**Path 1: Architecture & Design Mastery**

1. [Clean & Hexagonal Architecture Deep Dive](clean-architecture-guide.md)
2. [Domain-Driven Design Complete Guide](domain-driven-design-guide.md)
3. [Event-Driven Architecture Mastery](event-driven-architecture-guide.md)
4. [Design Patterns Decision Tree](design-patterns-decision-tree.md)

**Path 2: Java Fundamentals Mastery**

1. [Effective Java Generics](java-generics-guide.md)
2. [Java Optional Mastery](java-optional-guide.md)
3. [Java Functional Interfaces](java-functional-interfaces-guide.md)
4. [Java Enums Deep Dive](java-enum-guide.md)
5. [Modern Java Date/Time API](modern-java-datetime-guide.md)
6. [Exception Design and Handling](exception-design-guide.md)
7. [Modern Java Concurrency](modern-java-concurrency-guide.md)
8. [Java Concurrency Utilities](java-concurrency-utilities-guide.md)
9. [CompletableFuture Mastery](completable-future-guide.md)

**Path 3: Distributed Systems Developer**

1. [Apache Kafka Deep Dive](kafka-deep-dive-guide.md)
2. [Kafka Serialization & Deserialization](kafka-serialization-guide.md)
3. [Kafka Backpressure Handling](kafka-backpressure-guide.md)
4. [Kafka Zero Message Loss](kafka-zero-loss-guide.md)
5. [Apache Flink 2.0](apache-flink-guide.md)
6. [Flink Checkpointing and Operators](flink-checkpointing-guide.md)
7. [Event-Driven Architecture Mastery](event-driven-architecture-guide.md)
8. [Distributed Transactions & Consistency](distributed-transactions-consistency-guide.md)
9. [Modern Resilience with Resilience4j](resilience4j-complete-guide.md)
10. [API Gateway & Service Mesh Patterns](api-gateway-service-mesh-guide.md)
11. [Modern MongoDB Query Patterns](mongodb-query-guide.md)

**Path 4: Spring Framework Mastery**

1. [Spring Boot Production Mastery](spring-boot-production-guide.md)
2. [Spring Security Complete Guide](spring-security-guide.md)
3. [Spring WebFlux & Reactive Programming](spring-webflux-reactive-guide.md)
4. [Project Reactor](project-reactor-guide.md)

**Path 5: Quality & Best Practices**

1. [Design Patterns Decision Tree](design-patterns-decision-tree.md)
2. [Exception Design and Handling](exception-design-guide.md)
3. [Modern Java Unit Testing](modern-java-testing-guide.md)
4. [Modern Git Usage](git-modern-guide.md)

**Path 6: Data Engineering**

1. [Apache Flink 2.0](apache-flink-guide.md)
2. [Flink Checkpointing and Operators](flink-checkpointing-guide.md)
3. [Modern MongoDB Query Patterns](mongodb-query-guide.md)
4. [Modern Redis with Lettuce](redis-lettuce-guide.md)
5. [Database Migration & Schema Evolution](database-migration-guide.md)

**Path 7: Performance Engineering**

1. [JVM Performance Engineering](jvm-performance-engineering-guide.md)
2. [Performance Testing Mastery](performance-testing-guide.md)
3. [Java Application Profiling](java-profiling-guide.md)
4. [Modern Java Concurrency](modern-java-concurrency-guide.md)
5. [Apache Kafka Deep Dive](kafka-deep-dive-guide.md)
6. [Spring Boot Production Mastery](spring-boot-production-guide.md)

**Path 8: DevOps & SRE**

1. [Docker & Kubernetes for Java Services](docker-kubernetes-guide.md)
2. [Observability Engineering](observability-engineering-guide.md)
3. [Modern Resilience with Resilience4j](resilience4j-complete-guide.md)
4. [Chaos Engineering for Java](chaos-engineering-guide.md)
5. [Database Migration & Schema Evolution](database-migration-guide.md)

**Path 9: Security Engineering**

1. [Application Security Guide](application-security-guide.md)
2. [Secrets Management & Configuration](secrets-management-guide.md)
3. [Spring Security Complete Guide](spring-security-guide.md)
4. [Modern Resilience with Resilience4j](resilience4j-complete-guide.md)

---

## 🚀 Quick Start Recommendations

### I want to learn...

- **Java generics effectively** → Start with [Effective Java Generics](java-generics-guide.md)
- **Java Optional best practices** → Start with [Java Optional Mastery](java-optional-guide.md)
- **Functional interfaces & lambdas** → Start with [Java Functional Interfaces](java-functional-interfaces-guide.md)
- **Java enums and patterns** → Start with [Java Enums Deep Dive](java-enum-guide.md)
- **Clean/Hexagonal Architecture** → Start with [Clean Architecture Deep Dive](clean-architecture-guide.md)
- **Domain-Driven Design** → Start with [Domain-Driven Design Guide](domain-driven-design-guide.md)
- **Asynchronous programming** → Start with [CompletableFuture Mastery](completable-future-guide.md)
- **Virtual threads & modern concurrency** → Start with [Modern Java Concurrency](modern-java-concurrency-guide.md)
- **Java concurrency utilities** → Start with [Java Concurrency Utilities](java-concurrency-utilities-guide.md)
- **Reactive programming** → Start with [Spring WebFlux & Reactive](spring-webflux-reactive-guide.md)
- **Event-driven systems** → Start with [Event-Driven Architecture](event-driven-architecture-guide.md)
- **Distributed transactions & sagas** → Start with [Distributed Transactions & Consistency](distributed-transactions-consistency-guide.md)
- **REST API design** → Start with [REST API Design & Implementation](rest-api-design-guide.md)
- **API Gateway & Service Mesh** → Start with [API Gateway & Service Mesh Patterns](api-gateway-service-mesh-guide.md)
- **Apache Kafka internals** → Start with [Apache Kafka Deep Dive](kafka-deep-dive-guide.md)
- **Kafka serialization** → Start with [Kafka Serialization](kafka-serialization-guide.md)
- **Kafka backpressure & consumer lag** → Start with [Kafka Backpressure Handling](kafka-backpressure-guide.md)
- **Kafka zero message loss** → Start with [Kafka Zero Message Loss](kafka-zero-loss-guide.md)
- **Apache Flink stream processing** → Start with [Apache Flink 2.0](apache-flink-guide.md)
- **Flink checkpointing & fault tolerance** → Start with [Flink Checkpointing and Operators](flink-checkpointing-guide.md)
- **Project Reactor & reactive streams** → Start with [Project Reactor](project-reactor-guide.md)
- **Spring Boot production** → Start with [Spring Boot Production Mastery](spring-boot-production-guide.md)
- **Spring Security** → Start with [Spring Security Complete Guide](spring-security-guide.md)
- **Design patterns** → Start with [Design Patterns Decision Tree](design-patterns-decision-tree.md)
- **Testing best practices** → Start with [Modern Java Unit Testing](modern-java-testing-guide.md)
- **Building resilient systems** → Start with [Resilience4j Complete Guide](resilience4j-complete-guide.md)
- **Chaos engineering** → Start with [Chaos Engineering for Java](chaos-engineering-guide.md)
- **Application security** → Start with [Application Security Guide](application-security-guide.md)
- **Secrets management** → Start with [Secrets Management & Configuration](secrets-management-guide.md)
- **MongoDB optimization** → Start with [MongoDB Query Patterns](mongodb-query-guide.md)
- **Redis caching** → Start with [Redis with Lettuce](redis-lettuce-guide.md)
- **Database migrations** → Start with [Database Migration & Schema Evolution](database-migration-guide.md)
- **Docker & Kubernetes** → Start with [Docker & Kubernetes for Java](docker-kubernetes-guide.md)
- **Observability & monitoring** → Start with [Observability Engineering](observability-engineering-guide.md)
- **JVM performance tuning** → Start with [JVM Performance Engineering](jvm-performance-engineering-guide.md)
- **Load and performance testing** → Start with [Performance Testing Mastery](performance-testing-guide.md)
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
