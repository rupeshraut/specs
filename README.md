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

### ☕ Java Language & Core

- [**Modern Java Features Quick Reference**](modern-java-features-quick-reference.md)
  Concise lookup card for Java 17-24+ features: records, sealed types, pattern matching, virtual threads, and other modern language features.

- [**Java Streams & Functional Programming**](java-streams-functional-cheatsheet.md)
  Production patterns for Stream API, collectors, Optional, and functional composition for data transformation pipelines.

- [**Java Concurrency & Virtual Threads**](java-concurrency-virtual-threads-cheatsheet.md)
  Production patterns for concurrent programming with virtual threads, structured concurrency, and thread-safe design.

- [**Spring DI & Bean Lifecycle**](spring-di-bean-lifecycle-cheatsheet.md)
  Dependency injection patterns, bean lifecycle management, auto-configuration, and conditional wiring in Spring Boot 3.x.

### 💻 Development Best Practices

- [**Code Quality & Design**](code-quality-cheatsheet.md)
  Pre-coding checklist and reference guide. Covers SOLID principles, design patterns, refactoring, and clean code practices.

- [**Code Review Checklist**](code-review-checklist.md)
  Systematic checklist for reviewing enterprise Java code. Use as a reviewer to catch issues, or as an author for self-review before submitting a PR.

- [**Testing Strategy & Patterns**](testing-strategy-cheatsheet.md)
  Comprehensive testing patterns for distributed systems. Unit, integration, contract, and end-to-end testing with JUnit 5 and Testcontainers.

- [**GitHub Copilot Instructions**](github-copilot-enterprise-cheatsheet.md)
  Patterns for crafting effective Copilot instruction sets that enforce enterprise coding standards, architecture rules, and domain patterns.

### 🔧 Infrastructure & Operations

- [**Spring Boot Production-Ready**](spring-boot-production-cheatsheet.md)
  Production-hardened Spring Boot configuration, lifecycle management, and operational patterns for production deployments.

- [**Fault Tolerance & Resilience**](fault-tolerance-cheatsheet.md)
  Distributed systems resilience patterns using Resilience4j. Circuit breakers, retries, bulkheads, rate limiters, and fallback strategies.

- [**JVM Tuning & Performance**](jvm-tuning-performance-cheatsheet.md)
  JVM configuration, GC selection (G1GC, ZGC), profiling techniques, and container optimization for Java 21+.

- [**Observability**](observability-cheatsheet.md)
  Production-grade observability with metrics, logging, tracing, and alerting. Micrometer, Prometheus, Grafana, and OpenTelemetry patterns.

- [**Docker & Kubernetes Patterns**](docker-kubernetes-patterns-cheatsheet.md)
  Production container patterns for Java services. Dockerfiles, K8s manifests, health probes, resource management, and deployment strategies.

- [**Incident Response & Runbook Template**](incident-response-runbook-cheatsheet.md)
  Standardized incident response playbook, investigation framework, and runbook templates for distributed Java services.

### 📊 Data & Messaging

- [**Apache Kafka Patterns**](kafka-patterns-cheatsheet.md)
  Production-ready Kafka patterns for enterprise event-driven systems. Producers, consumers, topologies, retry strategies, and exactly-once semantics.

- [**MongoDB with Spring Data**](mongodb-spring-data-cheatsheet.md)
  MongoDB production patterns in Spring Boot. Queries, aggregations, indexing, transactions, and performance optimization.

- [**Distributed Locking & Coordination**](distributed-locking-cheatsheet.md)
  Production patterns for distributed locks, leader election, and coordination across service instances using MongoDB and Redis.

### 🛠️ Build, Deploy & Workflows

- [**Gradle Build & Plugin Patterns**](gradle-build-plugin-patterns-cheatsheet.md)
  Production Gradle patterns for multi-module builds, custom plugins, dependency management, and CI optimization with Kotlin DSL.

- [**Git Workflow & Branching Strategy**](git-workflow-branching-cheatsheet.md)
  Standardized Git workflow, branching strategy, commit conventions, and release management for enterprise teams.

### 📝 Documentation & Governance

- [**Architecture Decision Records (ADR) Template**](adr-template-cheatsheet.md)
  Standardized ADR format for documenting architecture decisions with examples from payment processing and distributed systems.

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

### By Development Phase

1. **Before designing a new service** → [Clean Architecture](clean-hexagonal-architecture-cheatsheet.md) + [Code Quality](code-quality-cheatsheet.md)
2. **Before designing an API** → [REST API Design](rest-api-design-cheatsheet.md) + [API Error Handling](api-error-handling-cheatsheet.md)
3. **Before applying a design pattern** → [Design Patterns Decision Guide](design-patterns-decision-guide.md)
4. **Before adding async communication** → [Event-Driven Architecture](event-driven-architecture-cheatsheet.md) + [Kafka Patterns](kafka-patterns-cheatsheet.md)
5. **Before writing concurrent code** → [Java Concurrency & Virtual Threads](java-concurrency-virtual-threads-cheatsheet.md)
6. **Before writing data pipelines** → [Java Streams & Functional](java-streams-functional-cheatsheet.md)
7. **Before configuring Spring** → [Spring DI & Bean Lifecycle](spring-di-bean-lifecycle-cheatsheet.md)
8. **Before setting up the build** → [Gradle Build Patterns](gradle-build-plugin-patterns-cheatsheet.md)
9. **Before writing tests** → [Testing Strategy](testing-strategy-cheatsheet.md)
10. **Before submitting a PR** → [Code Review Checklist](code-review-checklist.md)

### By Production Deployment

1. **Before containerizing** → [Docker & Kubernetes Patterns](docker-kubernetes-patterns-cheatsheet.md)
2. **Before going to production** → [Fault Tolerance](fault-tolerance-cheatsheet.md) + [Observability](observability-cheatsheet.md) + [Spring Boot Production](spring-boot-production-cheatsheet.md)
3. **Before handling incidents** → [Incident Response Runbook](incident-response-runbook-cheatsheet.md)
4. **Before performance tuning** → [JVM Tuning](jvm-tuning-performance-cheatsheet.md)

### By Use Case

- **Working with distributed systems** → [Distributed Locking](distributed-locking-cheatsheet.md)
- **Making architecture decisions** → [ADR Template](adr-template-cheatsheet.md)
- **Setting up team workflows** → [Git Workflow & Branching](git-workflow-branching-cheatsheet.md)
- **Using modern Java features** → [Modern Java Quick Reference](modern-java-features-quick-reference.md)
- **Configuring AI assistants** → [GitHub Copilot Instructions](github-copilot-enterprise-cheatsheet.md)

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
