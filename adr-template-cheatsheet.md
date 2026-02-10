# 📋 Architecture Decision Records (ADR) Template Cheat Sheet

> **Purpose:** Standardized ADR format for documenting architecture decisions with examples from payment processing and distributed systems.

---

## 📁 ADR File Structure

```
docs/adr/
├── 0001-use-mongodb-for-payment-storage.md
├── 0002-adopt-event-driven-architecture.md
├── 0003-outbox-pattern-for-reliable-events.md
├── 0004-kafka-over-jms-for-inter-service.md
├── 0005-hexagonal-architecture-for-payment-service.md
├── 0006-virtual-threads-for-request-handling.md
└── template.md
```

---

## 📝 ADR Template

```markdown
# ADR-NNNN: [Short Title in Imperative Mood]

**Status:** [Proposed | Accepted | Deprecated | Superseded by ADR-NNNN]
**Date:** YYYY-MM-DD
**Deciders:** [Names/roles]
**Technical Story:** [Ticket or epic reference]

## Context

What is the issue that we're seeing that motivates this decision?
What forces are at play (technical, business, organizational)?
Keep this factual and free of judgment.

## Decision

What is the change that we're proposing and/or doing?
State in active voice: "We will use X for Y."

## Consequences

### Positive
- What becomes easier?
- What improves?

### Negative
- What becomes harder?
- What trade-offs are we accepting?

### Risks
- What could go wrong?
- What mitigations are in place?

## Alternatives Considered

### Alternative 1: [Name]
Brief description. Why was it rejected?

### Alternative 2: [Name]
Brief description. Why was it rejected?

## References
- Links to relevant docs, RFCs, or previous ADRs
```

---

## 📖 Example ADRs

### ADR-0003: Outbox Pattern for Reliable Event Publishing

```markdown
# ADR-0003: Use Outbox Pattern for Reliable Event Publishing

**Status:** Accepted
**Date:** 2026-01-15
**Deciders:** Rupesh (Principal Eng), Team Lead
**Technical Story:** PAY-4521

## Context

Our payment service needs to persist payment state to MongoDB AND publish
domain events to Kafka. The dual-write problem means that if either operation
fails, we have data inconsistency: a payment saved without its event, or an
event published for an unsaved payment.

We require zero message loss for payment events as downstream services
(settlement, reconciliation, audit) depend on them.

## Decision

We will use the Transactional Outbox pattern:

1. Write the business document AND an outbox event to MongoDB in a single
   transaction
2. A background poller (leader-elected, single instance) reads unpublished
   outbox events and publishes to Kafka
3. On successful Kafka publish, mark the outbox event as published
4. A cleanup job removes published events older than 7 days

## Consequences

### Positive
- Zero message loss: events are persisted atomically with business data
- At-least-once delivery guarantee
- Survives Kafka downtime (events accumulate in outbox)
- Decouples business logic from messaging infrastructure

### Negative
- Increased write amplification (2 writes per operation instead of 1)
- Additional complexity: poller, leader election, cleanup
- Slight delay between business write and event publish (~100ms)
- Consumers MUST be idempotent (at-least-once means possible duplicates)

### Risks
- Outbox table growth if Kafka is down for extended period
  → Mitigation: monitoring on outbox.pending.count, alert at > 10,000
- Poller throughput may bottleneck under high load
  → Mitigation: batch polling (100 events/poll), monitor and tune

## Alternatives Considered

### Alternative 1: Dual Write (DB + Kafka separately)
Write to MongoDB, then publish to Kafka. Rejected because a crash between
the two operations loses the event. Unacceptable for payment events.

### Alternative 2: CDC (Change Data Capture) with MongoDB Change Streams
Use MongoDB Change Streams to react to document changes and publish events.
Rejected because: (a) change streams expose internal document structure,
not domain events; (b) harder to include business-meaningful event payloads;
(c) operational complexity of maintaining change stream resumability.

### Alternative 3: Event Sourcing
Store events as the source of truth, derive state from events. Rejected
as over-engineering for our current needs: most of our access patterns are
state-based reads, and the team lacks event sourcing experience.

## References
- Microservices Patterns (Chris Richardson), Ch. 3: Transactional Outbox
- ADR-0002: Adopt Event-Driven Architecture
```

### ADR-0006: Virtual Threads for Request Handling

```markdown
# ADR-0006: Adopt Virtual Threads for Request Handling

**Status:** Accepted
**Date:** 2026-02-01
**Deciders:** Rupesh (Principal Eng)
**Technical Story:** PAY-5102

## Context

Our payment service handles ~500 concurrent requests, each making 2-3
blocking I/O calls (MongoDB, payment gateway, Kafka). With platform threads
(Tomcat default pool of 200), we are regularly exhausting the thread pool
under peak load, causing request queuing and p99 latency spikes.

Options: increase thread pool (more memory), switch to reactive (WebFlux),
or adopt virtual threads (Java 21+).

## Decision

We will enable virtual threads in Spring Boot 3.2+:
  spring.threads.virtual.enabled: true

This applies virtual threads to: Tomcat request handling, @Async execution,
Kafka consumer threads, and scheduled tasks.

We will replace all synchronized blocks with ReentrantLock to avoid
carrier thread pinning.

## Consequences

### Positive
- Handles 10,000+ concurrent requests without thread pool exhaustion
- Zero code changes to business logic (drop-in replacement)
- Simpler mental model than reactive (imperative, blocking-style code)
- Dramatically reduced thread stack memory

### Negative
- Must audit all synchronized blocks → ReentrantLock migration
- ThreadLocal usage must be reviewed (memory per virtual thread)
- Debugging/profiling tools still catching up for virtual threads
- Team needs training on virtual thread specific pitfalls

### Risks
- Third-party libraries using synchronized internally could pin carriers
  → Mitigation: test under load, monitor pinned thread events via JFR
- Unbounded virtual threads could overwhelm downstream resources
  → Mitigation: Semaphore to limit concurrent DB/gateway calls

## Alternatives Considered

### Alternative 1: Increase Tomcat thread pool to 500
Quick fix, but doubles memory consumption for thread stacks and doesn't
scale beyond ~1000 concurrent connections.

### Alternative 2: Migrate to Spring WebFlux (reactive)
Maximum scalability, but requires rewriting all service code to reactive
chains. Team has limited reactive experience. Debugging is significantly
harder. Estimated 3-month migration effort.

## References
- JEP 444: Virtual Threads
- Spring Boot 3.2 Virtual Threads Support
- ADR-0005: Hexagonal Architecture (adapter layer isolates framework)
```

---

## ✅ ADR Best Practices

```
1.  ONE decision per ADR — don't bundle multiple decisions.
2.  IMMUTABLE once accepted — supersede, don't edit.
3.  CONTEXT is the most important section — capture WHY, not just WHAT.
4.  ALTERNATIVES must be genuine — show you considered other options.
5.  CONSEQUENCES are honest — include negatives and risks.
6.  NUMBER sequentially — 0001, 0002, ... easy to reference.
7.  STATUS is maintained — mark deprecated/superseded decisions.
8.  WRITE for the future reader — they don't have your current context.
9.  LINK between ADRs — decisions build on each other.
10. REVIEW as a team — ADRs are shared understanding, not individual notes.
```

---

*Last updated: February 2026*
