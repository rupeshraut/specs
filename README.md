# 📚 Enterprise Java Development Cheatsheets

A collection of production-ready cheatsheets for building scalable, resilient, distributed systems with Java 21+, Spring Boot 3.x, Kafka, MongoDB, and modern cloud infrastructure.

---

## 📖 Table of Contents

### 🏛️ Architecture & Design Patterns

- [**Clean & Hexagonal Architecture**](clean-hexagonal-architecture-cheatsheet.md)
  Practical guide to structuring enterprise applications with Clean/Hexagonal Architecture. Package structure, dependency rules, and modular design.

- [**Event-Driven Architecture Patterns**](event-driven-architecture-cheatsheet.md)
  Production-grade patterns for event-driven systems. Sagas, event sourcing, eventual consistency, and async processing patterns.

- [**Design Patterns Decision Guide**](design-patterns-decision-guide.md)
  When to use (and when NOT to use) each design pattern, with modern Java 21+ implementations using records, sealed types, and pattern matching.

### 🌐 API Design

- [**REST API Design**](rest-api-design-cheatsheet.md)
  Production-grade REST API design patterns for enterprise services. Endpoint design, versioning strategies, and best practices with Spring Web.

- [**API Error Handling & Resilience**](api-error-handling-cheatsheet.md)
  Production patterns for handling errors, retries, rate limiting, and graceful degradation at the API layer.

### 💻 Development Best Practices

- [**Code Quality & Design**](code-quality-cheatsheet.md)
  Pre-coding checklist and reference guide. Covers SOLID principles, design patterns, refactoring, and clean code practices.

- [**Code Review Checklist**](code-review-checklist.md)
  Systematic checklist for reviewing enterprise Java code. Use as a reviewer to catch issues, or as an author for self-review before submitting a PR.

- [**Testing Strategy & Patterns**](testing-strategy-cheatsheet.md)
  Comprehensive testing patterns for distributed systems. Unit, integration, contract, and end-to-end testing with JUnit 5 and Testcontainers.

### 🔧 Infrastructure & Operations

- [**Spring Boot Production-Ready**](spring-boot-production-cheatsheet.md)
  Production-hardened Spring Boot configuration, lifecycle management, and operational patterns for production deployments.

- [**Fault Tolerance & Resilience**](fault-tolerance-cheatsheet.md)
  Distributed systems resilience patterns using Resilience4j. Circuit breakers, retries, bulkheads, rate limiters, and fallback strategies.

- [**JVM Tuning & Performance**](jvm-tuning-performance-cheatsheet.md)
  JVM configuration, GC selection (G1GC, ZGC), profiling techniques, and container optimization for Java 21+.

- [**Observability**](observability-cheatsheet.md)
  Production-grade observability with metrics, logging, tracing, and alerting. Micrometer, Prometheus, Grafana, and OpenTelemetry patterns.

### 📊 Data & Messaging

- [**Apache Kafka Patterns**](kafka-patterns-cheatsheet.md)
  Production-ready Kafka patterns for enterprise event-driven systems. Producers, consumers, topologies, retry strategies, and exactly-once semantics.

- [**MongoDB with Spring Data**](mongodb-spring-data-cheatsheet.md)
  MongoDB production patterns in Spring Boot. Queries, aggregations, indexing, transactions, and performance optimization.

---

## 🎯 How to Use These Cheatsheets

Each cheatsheet follows a consistent structure:

1. **Decision Framework** — Questions to answer before implementing
2. **Patterns & Examples** — Production-ready code snippets
3. **Common Pitfalls** — Anti-patterns and what to avoid
4. **Production Checklist** — Pre-deployment verification

### Stack Context

All cheatsheets are designed for:
- **Language:** Java 21+
- **Framework:** Spring Boot 3.x
- **Database:** MongoDB 7.x
- **Messaging:** Apache Kafka, ActiveMQ Artemis
- **Resilience:** Resilience4j
- **Observability:** Micrometer, Prometheus, OpenTelemetry
- **Testing:** JUnit 5, Testcontainers
- **Deployment:** Docker, Kubernetes

---

## 🚀 Quick Start

1. **Before designing a new service** → Read [Clean Architecture](clean-hexagonal-architecture-cheatsheet.md) and [Code Quality](code-quality-cheatsheet.md)
2. **Before designing an API** → Read [REST API Design](rest-api-design-cheatsheet.md) and [API Error Handling](api-error-handling-cheatsheet.md)
3. **Before applying a design pattern** → Read [Design Patterns Decision Guide](design-patterns-decision-guide.md)
4. **Before adding async communication** → Read [Event-Driven Architecture](event-driven-architecture-cheatsheet.md) and [Kafka Patterns](kafka-patterns-cheatsheet.md)
5. **Before submitting a PR** → Read [Code Review Checklist](code-review-checklist.md)
6. **Before going to production** → Read [Fault Tolerance](fault-tolerance-cheatsheet.md), [Observability](observability-cheatsheet.md), and [Spring Boot Production](spring-boot-production-cheatsheet.md)
7. **Before performance tuning** → Read [JVM Tuning](jvm-tuning-performance-cheatsheet.md) and [MongoDB Optimization](mongodb-spring-data-cheatsheet.md)
8. **Before writing tests** → Read [Testing Strategy](testing-strategy-cheatsheet.md)

---

## 📝 Contributing

These cheatsheets are living documents. Update them as:
- New patterns emerge
- Technology versions change
- Production lessons are learned
- Team standards evolve

---

## 📄 License

Internal reference documentation for enterprise Java development.
