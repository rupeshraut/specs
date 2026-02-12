# Event Bus Connector — Technical Design Document

**Version:** 1.0.0  
**Status:** Draft  
**Author:** Engineering  
**Date:** February 2026  
**Stack:** Java 21, Spring Boot 3.4, Spring Kafka, Resilience4j, Micrometer  
**Build:** Gradle 8.x

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Design Principles](#2-design-principles)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Module Design](#4-module-design)
   - 4.1 [Configuration Model](#41-configuration-model)
   - 4.2 [Dynamic Container Registry & Factory](#42-dynamic-container-registry--factory)
   - 4.3 [Lifecycle Manager](#43-lifecycle-manager)
   - 4.4 [Listener Adapter Layer](#44-listener-adapter-layer)
   - 4.5 [Retry & Recovery Engine (Resilience4j)](#45-retry--recovery-engine-resilience4j)
   - 4.6 [Tiered Retry Architecture](#46-tiered-retry-architecture)
   - 4.7 [Dead Letter Topic (DLT) Handling](#47-dead-letter-topic-dlt-handling)
   - 4.8 [Offset Management](#48-offset-management)
   - 4.9 [Rebalance Prevention](#49-rebalance-prevention)
   - 4.10 [Deserialization Failure Handling](#410-deserialization-failure-handling)
   - 4.11 [Graceful Shutdown](#411-graceful-shutdown)
   - 4.12 [Header Propagation & Enrichment](#412-header-propagation--enrichment)
   - 4.13 [Observability Layer](#413-observability-layer)
   - 4.14 [Admin API](#414-admin-api)
5. [Key Interfaces](#5-key-interfaces)
6. [Threading Model](#6-threading-model)
7. [Configuration Validation](#7-configuration-validation)
8. [Idempotency Support](#8-idempotency-support)
9. [Multi-Cluster Support](#9-multi-cluster-support)
10. [Testing Strategy](#10-testing-strategy)
11. [Package Structure](#11-package-structure)
12. [Technology Matrix](#12-technology-matrix)
13. [Appendix: Configuration Reference](#appendix-configuration-reference)

---

## 1. Executive Summary

The Event Bus Connector is a production-grade abstraction over Spring Kafka that provides zero-message-loss semantics, tiered retry with Dead Letter Topic (DLT) recovery, dynamic listener container management, full consumer lifecycle control, and resilience via Resilience4j. The user's sole responsibility is implementing a record handler — the connector owns everything else: offset management, retry orchestration, circuit breaking, DLQ publishing, observability, and lifecycle coordination.

**Non-functional guarantees:**

- Zero message loss under all failure modes (consumer crash, downstream outage, partial batch failure, rebalance, poison pill).
- Partition ordering preserved within the main consumer. Ordering relaxed in retry tiers (configurable strict mode available).
- At-least-once delivery semantics with optional idempotency support for effective exactly-once.

---

## 2. Design Principles

1. **Single Responsibility:** Each module owns one concern — no god classes.
2. **Fail Fast at Startup:** Configuration is validated for mathematical coherence (retry budgets vs. poll intervals) before any container starts.
3. **User Sees Only Handlers:** The connector's internal machinery (retries, offsets, DLQ, lifecycle) is invisible to the user. They implement a handler interface and receive records.
4. **Composition Over Inheritance:** Resilience decorators (Retry, CircuitBreaker) are composed around handler invocations, not baked into base classes.
5. **Modern Java:** Records for immutable config, sealed interfaces where appropriate, pattern matching, virtual threads awareness.
6. **Observable by Default:** Every container, every tier, every circuit breaker emits metrics and structured logs without opt-in.

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Event Bus Connector                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────┐  ┌───────────────┐  ┌─────────────────────────┐ │
│  │  Configuration  │  │  Lifecycle    │  │  Dynamic Container      │ │
│  │  (Properties +  │  │  Manager      │  │  Registry               │ │
│  │   Validation)   │  │               │  │                         │ │
│  └───────┬────────┘  └───────┬───────┘  └────────────┬────────────┘ │
│          │                   │                        │              │
│  ┌───────▼───────────────────▼────────────────────────▼────────────┐ │
│  │                    Container Factory                             │ │
│  │   Creates ConcurrentMessageListenerContainer per topic          │ │
│  │   + Retry Tier containers + wires error handlers                │ │
│  └──────────────────────────┬──────────────────────────────────────┘ │
│                              │                                       │
│  ┌──────────────────────────▼──────────────────────────────────────┐ │
│  │                    Listener Adapter Layer                        │ │
│  │  ┌───────────────────────┐  ┌─────────────────────────────────┐ │ │
│  │  │  Single Record        │  │  Batch Record                   │ │ │
│  │  │  Acknowledging        │  │  Acknowledging                  │ │ │
│  │  │  Listener Adapter     │  │  Listener Adapter               │ │ │
│  │  │                       │  │  (partial failure aware)        │ │ │
│  │  └───────────┬───────────┘  └──────────────┬──────────────────┘ │ │
│  └──────────────┼─────────────────────────────┼────────────────────┘ │
│                  │                             │                      │
│  ┌──────────────▼─────────────────────────────▼────────────────────┐ │
│  │                  Retry & Recovery Engine                         │ │
│  │  ┌──────────────┐ ┌────────────────┐ ┌────────────────────────┐ │ │
│  │  │ Resilience4j │ │ Partition-Aware │ │ Exception              │ │ │
│  │  │ Retry +      │ │ Offset Manager │ │ Classifier             │ │ │
│  │  │ CircuitBreaker│ │               │ │                        │ │ │
│  │  └──────────────┘ └────────────────┘ └────────────────────────┘ │ │
│  └─────────────────────────────┬───────────────────────────────────┘ │
│                                │                                     │
│  ┌─────────────────────────────▼───────────────────────────────────┐ │
│  │                  Tiered Retry Pipeline                           │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐ │ │
│  │  │ Tier 0   │  │ Tier 1   │  │ Tier 2   │  │ Tier 3         │ │ │
│  │  │ In-Memory│→ │ 10s delay│→ │ 60s delay│→ │ 300s delay     │ │ │
│  │  │ R4j Retry│  │ Topic    │  │ Topic    │  │ Topic          │ │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └───────┬────────┘ │ │
│  │                                                      │         │ │
│  │                                              ┌───────▼───────┐ │ │
│  │                                              │  DLT Handler  │ │ │
│  │                                              └───────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                  Observability Layer                             │ │
│  │  Micrometer metrics · Structured logging · Health indicator     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Module Design

### 4.1 Configuration Model

A strongly-typed configuration hierarchy using `@ConfigurationProperties("eventbus")`.

**Root configuration** holds a `Map<String, TopicBindingProperties>` where the key is the logical binding name.

**TopicBindingProperties** contains:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `topic` | `String` | — | Kafka topic name (required) |
| `groupId` | `String` | — | Consumer group ID (required) |
| `listenerType` | `ListenerType` | `SINGLE` | SINGLE or BATCH |
| `ackMode` | `AckMode` | `MANUAL_IMMEDIATE` | MANUAL or MANUAL_IMMEDIATE |
| `concurrency` | `int` | `1` | Number of consumer threads |
| `maxPollRecords` | `int` | `50` (single), `200` (batch) | Records per poll |
| `maxPollIntervalMs` | `long` | `600000` | Max poll interval in ms |
| `sessionTimeoutMs` | `long` | `45000` | Session timeout |
| `heartbeatIntervalMs` | `long` | `10000` | Heartbeat interval |
| `autoStartup` | `boolean` | `true` | Start container on registration |
| `staticMembership` | `boolean` | `false` | Enable static group membership |
| `cluster` | `String` | `default` | Cluster alias for multi-cluster |
| `retry` | `RetryProperties` | — | Retry configuration |
| `circuitBreaker` | `CircuitBreakerProperties` | — | Circuit breaker configuration |
| `dlt` | `DltProperties` | — | Dead letter topic configuration |
| `batchFailureStrategy` | `BatchFailureStrategy` | `SEEK_TO_FAILED` | SEEK_TO_FAILED or DLQ_AND_CONTINUE |
| `orderingMode` | `OrderingMode` | `RELAXED_IN_RETRY` | RELAXED_IN_RETRY or STRICT |

**RetryProperties** (nested):

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `tier0.maxAttempts` | `int` | `3` | In-memory retry attempts |
| `tier0.initialBackoffMs` | `long` | `100` | Initial backoff |
| `tier0.multiplier` | `double` | `2.0` | Backoff multiplier |
| `tier0.maxBackoffMs` | `long` | `2000` | Backoff ceiling |
| `tiers` | `List<RetryTierProperties>` | 3 default tiers | Topic-based retry tiers |
| `retryableExceptions` | `List<Class>` | all | Exceptions eligible for retry |
| `nonRetryableExceptions` | `List<Class>` | empty | Exceptions sent directly to DLT |
| `skipToTierMapping` | `Map<Class, Integer>` | empty | Exception → tier number |

**RetryTierProperties** (per tier):

| Property | Type | Default |
|----------|------|---------|
| `delayMs` | `long` | tier-dependent |
| `maxAttempts` | `int` | `3` |
| `topicSuffix` | `String` | `retry-{n}` |
| `concurrency` | `int` | `1` |

**CircuitBreakerProperties** (nested):

| Property | Type | Default |
|----------|------|---------|
| `enabled` | `boolean` | `false` |
| `failureRateThreshold` | `float` | `50.0` |
| `slowCallRateThreshold` | `float` | `80.0` |
| `slowCallDurationMs` | `long` | `5000` |
| `waitDurationInOpenStateMs` | `long` | `60000` |
| `slidingWindowSize` | `int` | `100` |
| `minimumNumberOfCalls` | `int` | `10` |
| `permittedNumberOfCallsInHalfOpenState` | `int` | `5` |
| `automaticPauseOnOpen` | `boolean` | `true` |

**DltProperties** (nested):

| Property | Type | Default |
|----------|------|---------|
| `topicSuffix` | `String` | `DLT` |
| `strategy` | `DltStrategy` | `STORE_ONLY` |
| `replay.maxReplays` | `int` | `3` |
| `replay.cooldownMs` | `long` | `3600000` |
| `replay.batchSize` | `int` | `100` |

**Enums:**

```java
public enum ListenerType { SINGLE, BATCH }

public enum BatchFailureStrategy { SEEK_TO_FAILED, DLQ_AND_CONTINUE }

public enum OrderingMode { RELAXED_IN_RETRY, STRICT }

public enum DltStrategy { STORE_ONLY, ALERT_AND_STORE, SCHEDULED_REPLAY, CUSTOM }

public enum ContainerState { CREATED, RUNNING, PAUSED, STOPPED, DESTROYED }

public enum ExceptionRouting { NEXT_TIER, SKIP_TO_TIER, DEAD_LETTER, PAUSE_CONTAINER }
```

---

### 4.2 Dynamic Container Registry & Factory

#### Container Factory

A Spring `@Component` that produces a fully-wired `ConcurrentMessageListenerContainer` given a `TopicBindingProperties` configuration.

**Factory workflow:**

```
TopicBindingProperties
       │
       ▼
┌─────────────────────────────────┐
│  1. Build ConsumerFactory       │ → DefaultKafkaConsumerFactory
│     - ErrorHandlingDeserializer │   wrapping actual deserializers
│     - enable.auto.commit=false  │
│     - isolation.level=          │
│       read_committed            │
│     - max.poll.records          │
│     - session/heartbeat/poll    │
│       interval properties       │
│     - group.instance.id         │
│       (if static membership)    │
│                                 │
│  2. Build ContainerProperties   │ → AckMode from config
│     - Ack mode                  │   (MANUAL_IMMEDIATE default)
│     - Poll timeout              │
│     - Idle between polls        │
│     - Rebalance listener        │
│                                 │
│  3. Wire Listener Adapter       │ → Single or Batch acknowledging
│     - Wraps user handler        │   listener based on ListenerType
│     - Decorates with R4j        │
│                                 │
│  4. Wire Error Handler          │ → Custom CommonErrorHandler with
│     - Partial batch offset mgmt │   retry + DLQ + offset management
│                                 │
│  5. Create Retry Tier           │ → One container per retry tier
│     Containers                  │   with delay-based listeners
│                                 │
│  6. Register all containers     │ → Store in Container Registry
└─────────────────────────────────┘
```

**Key design decisions:**

- The factory accepts an explicit `ConsumerFactory` for multi-cluster support rather than always using a single autowired instance.
- Retry tier containers are created as part of the same factory call — they share the circuit breaker instance with the main container.
- Each container gets a unique bean name: `eventbus-{bindingName}` for main, `eventbus-{bindingName}-retry-{n}` for tiers.

#### Container Registry

A `ConcurrentHashMap<String, ContainerRegistration>` keyed by binding name.

**ContainerRegistration** is a record holding:

```java
public record ContainerRegistration(
    String bindingName,
    TopicBindingProperties config,
    ConcurrentMessageListenerContainer<?, ?> mainContainer,
    List<ConcurrentMessageListenerContainer<?, ?>> retryTierContainers,
    CircuitBreaker circuitBreaker,       // shared across main + tier containers
    ContainerState state
) {}
```

**Registry operations:**

| Method | Description |
|--------|-------------|
| `register(ContainerRegistration)` | Add a new registration. Throws if binding name exists. |
| `unregister(String bindingName)` | Remove and return the registration. Does not stop/destroy. |
| `get(String bindingName)` | Lookup by binding name. Returns `Optional`. |
| `getByTopic(String topic)` | Lookup by topic name. |
| `listAll()` | Returns unmodifiable view of all registrations. |
| `updateState(String, ContainerState)` | Atomically update the state field. |

---

### 4.3 Lifecycle Manager

A dedicated component wrapping the container registry with a clean command API for lifecycle operations.

**State machine per container:**

```
CREATED ──start()──► RUNNING ──pause()──► PAUSED
   ▲                    │    ◄──resume()──    │
   │                    │                     │
   │               stop()│               stop()│
   │                    ▼                     ▼
   │                STOPPED ◄────────────STOPPED
   │                    │
   │              destroy()
   │                    ▼
   └─ (removed)     DESTROYED
```

**Operations:**

| Command | Behavior |
|---------|----------|
| `start(bindingName)` | Validates CREATED or STOPPED → RUNNING. Calls `container.start()` on main + all retry tier containers. |
| `stop(bindingName)` | Validates RUNNING or PAUSED → STOPPED. Gracefully finishes in-flight processing. Calls `container.stop()` on all containers. |
| `pause(bindingName)` | Validates RUNNING → PAUSED. Calls `container.pause()`. Consumer stays in group, stops fetching. |
| `resume(bindingName)` | Validates PAUSED → RUNNING. Calls `container.resume()`. Fetching resumes from last committed offset. |
| `destroy(bindingName)` | Calls `stop()` then `container.destroy()`. Removes from registry. Consumer leaves group, triggers rebalance. |

**Concurrency safety:** All operations use a `ConcurrentHashMap<String, ReentrantLock>` (striped lock per binding name). Lifecycle commands for different bindings don't block each other.

**Invalid transition handling:** Throws a domain-specific `InvalidContainerStateException` with the current state and the attempted operation, e.g.: `"Cannot resume binding 'orders': current state is STOPPED, resume requires PAUSED"`.

**Cascade behavior:**

- `pause(bindingName)` pauses the main container and all its retry tier containers.
- `resume(bindingName)` resumes all containers in the binding.
- `destroy(bindingName)` destroys all containers and unregisters the binding.

---

### 4.4 Listener Adapter Layer

Two listener implementations, both operating in acknowledging mode.

#### 4.4.1 Single Record Acknowledging Listener

Implements `AcknowledgingMessageListener<K, V>`.

**Processing flow:**

```
Spring delivers one record
       │
       ▼
┌─────────────────────────────────────┐
│  1. Check deserialization headers   │
│     ├─ Poison pill? → DLQ + ack    │
│     └─ OK → continue               │
│                                     │
│  2. Check poll-interval safety      │
│     valve (elapsed time check)      │
│                                     │
│  3. Invoke user handler via         │
│     Resilience4j decoration chain:  │
│       CircuitBreaker                │
│         └─ Retry                    │
│              └─ UserHandler.handle()│
│                                     │
│  4. Outcome:                        │
│     ├─ Success                      │
│     │   └─ ack.acknowledge()        │
│     │       MANUAL_IMMEDIATE: sync  │
│     │       MANUAL: batched commit  │
│     │                               │
│     ├─ Retries exhausted            │
│     │   └─ Exception classifier:    │
│     │       ├─ NON_RETRYABLE → DLT  │
│     │       ├─ NEXT_TIER → Tier 1   │
│     │       └─ SKIP_TO_TIER → Tier N│
│     │   └─ ack.acknowledge()        │
│     │                               │
│     └─ Circuit breaker OPEN         │
│         └─ Pause container          │
│            (no ack, no DLQ)         │
└─────────────────────────────────────┘
```

**Ordering guarantee:** Spring Kafka delivers records sequentially per partition within a single consumer thread. Retries happen in-line on the consumer thread (blocking). The next record in the partition is not processed until the current one completes or is routed.

#### 4.4.2 Batch Acknowledging Listener — Partial Failure Handling

Implements `BatchAcknowledgingMessageListener<K, V>`.

**Strategy A: SEEK_TO_FAILED (default — zero loss, no duplicates)**

```
Poll returns batch of N records (partition-ordered)
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│  Start wall-clock timer                                      │
│                                                              │
│  Track: Map<TopicPartition, OffsetAndMetadata> successOffsets│
│                                                              │
│  For each record[i] (sequential, offset-ordered):            │
│                                                              │
│    1. Poll-interval safety check:                            │
│       elapsed > 70% of max.poll.interval.ms?                 │
│       ├─ YES → Commit successOffsets, seek to record[i],     │
│       │        return (triggers next poll)                    │
│       └─ NO → continue                                       │
│                                                              │
│    2. Check deserialization headers (poison pill check)       │
│                                                              │
│    3. Invoke user handler via R4j decoration chain            │
│                                                              │
│    4. Outcome:                                               │
│       ├─ Success → Track offset in successOffsets            │
│       │                                                      │
│       └─ Failure (retries exhausted):                        │
│            ┌─ SEEK_TO_FAILED strategy:                       │
│            │  a. commitSync(successOffsets)                   │
│            │     → commits ALL successfully processed records│
│            │  b. seek(failedPartition, failedOffset)          │
│            │     → rewind to first failure point             │
│            │  c. For other partitions: seek to last           │
│            │     committed + 1                                │
│            │  d. Throw BatchProcessingException               │
│            │     → container re-polls from failure point     │
│            │  e. No duplicates: committed records won't       │
│            │     be re-delivered                              │
│            └─────────────────────────────────────────────────│
│                                                              │
│  All records processed successfully:                         │
│    → ack.acknowledge() for full batch                        │
└──────────────────────────────────────────────────────────────┘
```

**Strategy B: DLQ_AND_CONTINUE (higher throughput, relaxed ordering)**

Failed records go to DLQ immediately, remaining records continue processing. The full batch is committed at the end.

**Custom `CommonErrorHandler` implementation** for SEEK_TO_FAILED:

1. Receives the `BatchProcessingException` (custom exception wrapping the failed index and the consumer records).
2. Extracts the failed index from the exception.
3. Builds `Map<TopicPartition, OffsetAndMetadata>` for all records before the failed index.
4. Calls `consumer.commitSync(offsetMap)`.
5. Calls `consumer.seek(partition, failedOffset)` for the failed partition.
6. For other partitions with records after the abort point, seeks back to last committed offset + 1.

This gives **commit-the-good, replay-from-the-bad** semantics.

---

### 4.5 Retry & Recovery Engine (Resilience4j)

We intentionally bypass Spring Kafka's built-in `DefaultErrorHandler` retries. Instead, the user's handler invocation is wrapped with composable Resilience4j decorators.

**Decoration chain (innermost to outermost):**

```
UserHandler.handle(record)
       │
       ▼
Resilience4j Retry (Tier 0 — in-memory)
  - Configurable: max attempts, backoff strategy
  - Retryable exceptions whitelist
  - Backoff: exponential with jitter
       │
       ▼
Resilience4j CircuitBreaker
  - Protects downstream dependencies
  - When OPEN → fast-fail, no retries burned
  - Shared across all consumer threads for a binding
       │
       ▼
Listener Adapter
```

**Why Retry inside CircuitBreaker:**

- The Retry wraps the handler directly — transient failures get retried with backoff.
- The CircuitBreaker wraps the retry — if a downstream system is truly down, the circuit opens and retries stop. This prevents burning retry budgets against an unreachable system.

**CircuitBreaker ↔ Container lifecycle integration:**

```
CircuitBreaker state change event
       │
       ├─ CLOSED → OPEN:
       │    Lifecycle Manager pauses the binding
       │    Log warning, emit metric
       │    Records stop being consumed — no loss, no DLQ
       │
       ├─ OPEN → HALF_OPEN:
       │    Lifecycle Manager resumes the binding
       │    A few records flow to test the circuit
       │
       └─ HALF_OPEN → CLOSED:
            Normal processing resumes
            Emit recovery metric
```

This prevents message loss during outages — records that would fail only because a dependency is down are not consumed, not retried, and not DLQ'd. We pause, wait, and resume.

**Instance scoping:**

| Component | Scope | Thread Safety |
|-----------|-------|---------------|
| `CircuitBreaker` | One per binding, shared across consumer threads | Thread-safe by design |
| `Retry` | Configuration per binding, new context per invocation | Stateless per call |

---

### 4.6 Tiered Retry Architecture

#### The Problem with Flat Retry

Failures have different recovery timescales:

| Failure Type | Recovery Time | Example |
|-------------|---------------|---------|
| Transient | Milliseconds to seconds | Connection timeout, optimistic lock |
| Short-lived | Seconds to minutes | Downstream restart, brief rate limiting |
| Extended | Minutes to hours | Database failover, third-party outage |
| Permanent | Never | Schema mismatch, business rule violation |

Flat retry either retries too aggressively or too conservatively. Tiered retry matches retry intensity to failure severity.

#### Tier Pipeline

```
Original Topic: orders.events
          │
          ▼
┌─────────────────────┐
│  Main Consumer      │
│  Tier 0: In-memory  │  3 attempts, 100ms/200ms/400ms (Resilience4j)
└─────────┬───────────┘
          │ Exhausted
          ▼
┌─────────────────────┐
│  Retry Topic Tier 1 │  orders.events.retry-1
│  Delay: 10 seconds  │  3 attempts per delivery (each with Tier 0 in-memory retry)
│  Max attempts: 3    │
└─────────┬───────────┘
          │ Exhausted
          ▼
┌─────────────────────┐
│  Retry Topic Tier 2 │  orders.events.retry-2
│  Delay: 60 seconds  │  3 attempts per delivery
│  Max attempts: 3    │
└─────────┬───────────┘
          │ Exhausted
          ▼
┌─────────────────────┐
│  Retry Topic Tier 3 │  orders.events.retry-3
│  Delay: 300 seconds │  3 attempts per delivery
│  Max attempts: 3    │
└─────────┬───────────┘
          │ Exhausted
          ▼
┌─────────────────────┐
│  Dead Letter Topic  │  orders.events.DLT
│  (Terminal)         │
└─────────────────────┘
```

**Total attempts across all tiers:**

```
Tier 0: 3
Tier 1: 3 deliveries × 3 in-memory retries = 9
Tier 2: 3 × 3 = 9
Tier 3: 3 × 3 = 9
─────────────────
Grand total: 30 attempts over ~18 minutes
```

#### Why Separate Topics Per Tier

- Each tier has a fixed delay — no per-record delay scheduling.
- Independent concurrency, monitoring, and alerting per tier.
- Each tier's consumer can be paused independently.
- Tier 3 filling up is a visible signal of extended outages.

#### Delay Mechanism — Partition Pause/Resume

Kafka has no native delayed delivery. The connector uses a timestamp-based pause approach:

```
Retry tier consumer polls a batch
       │
       ▼
For each record:
  Extract x-eventbus-retry-timestamp header
  eligibleTime = retryTimestamp + tierDelay
       │
       ├─ now() ≥ eligibleTime → Process the record
       │    ├─ Success → Acknowledge
       │    └─ Failure → Publish to next tier (or DLT)
       │
       └─ now() < eligibleTime → Not yet eligible
            ├─ PAUSE the partition
            ├─ Schedule delayed task to RESUME after (eligibleTime - now())
            └─ Return from listener (commit only processed records)
```

Since records within a partition are ordered by publish time and all records in a tier share the same delay, once we hit a not-yet-eligible record, all subsequent records in that partition are also ineligible. Pausing at the first ineligible record is correct and efficient.

#### Exception Classifier — Smart Routing

```
┌──────────────────────────────────────────────────────┐
│              Exception Classifier                     │
│                                                      │
│  Retryable (transient):                              │
│    ConnectionTimeout, SocketException,               │
│    OptimisticLockException                           │
│    → NEXT_TIER: follow the tier chain                │
│                                                      │
│  Retryable (extended):                               │
│    ServiceUnavailableException,                      │
│    CircuitBreakerOpenException                       │
│    → SKIP_TO_TIER: jump to Tier 2 or 3              │
│                                                      │
│  Non-retryable (permanent):                          │
│    DeserializationException,                         │
│    SchemaValidationException,                        │
│    BusinessRuleViolationException                    │
│    → DEAD_LETTER: bypass all tiers, go to DLT       │
│                                                      │
│  Unknown:                                            │
│    → NEXT_TIER (conservative default)                │
└──────────────────────────────────────────────────────┘
```

Configured per binding as a map of exception class → routing strategy.

#### Retry Tier Container Management

```
TopicBinding: "orders" (main)
  │
  │  Container Factory auto-creates:
  │
  ├── Container: orders.events            (main, concurrency=3)
  ├── Container: orders.events.retry-1    (tier 1, concurrency=1)
  ├── Container: orders.events.retry-2    (tier 2, concurrency=1)
  ├── Container: orders.events.retry-3    (tier 3, concurrency=1)
  └── (DLT has no consumer unless explicitly registered)
```

Retry tier concurrency is intentionally low — retry traffic should be low-volume, and lower concurrency reduces load on recovering downstream systems.

**Lifecycle coupling:** When the main container is paused (circuit breaker opened), all retry tier containers are also paused. When destroyed, all tiers are destroyed. They share the circuit breaker.

#### Ordering in Tiered Retry

Once a record leaves the main topic and enters a retry tier, ordering relative to other records in the original partition is lost. This is inherent to the pattern.

**Mitigation options (configurable per binding):**

| Mode | Behavior |
|------|----------|
| `RELAXED_IN_RETRY` (default) | Retry records may be processed after their successors. Records use the same key for partition affinity within retry topics. |
| `STRICT` | Tiered retry is disabled. Only Tier 0 (in-memory) is used. On exhaustion, either DLT the record or pause the partition until manual intervention. Blocks the partition. |

---

### 4.7 Dead Letter Topic (DLT) Handling

#### DLT Publisher

A dedicated `KafkaTemplate` configured for zero-loss publishing:

- `acks=all`
- `enable.idempotence=true`
- `retries=Integer.MAX_VALUE`
- `max.in.flight.requests.per.connection=1`

Publishing uses **synchronous send with `get()`** — if the DLT publish fails, the original record is not acknowledged. This upholds the zero-loss guarantee.

#### DLT Record Structure

```
Key:     Original record key (preserved for partitioning affinity)
Value:   Original record value (byte-perfect copy)
Headers:
  x-eventbus-original-topic
  x-eventbus-original-partition
  x-eventbus-original-offset
  x-eventbus-original-timestamp
  x-eventbus-dlt-reason:           RETRIES_EXHAUSTED | NON_RETRYABLE | MANUAL
  x-eventbus-dlt-timestamp
  x-eventbus-total-attempts
  x-eventbus-first-failure-timestamp
  x-eventbus-last-exception-class
  x-eventbus-last-exception-message
  x-eventbus-last-exception-stacktrace   (truncated to 2KB)
  x-eventbus-retry-tier:                 last tier attempted
  x-eventbus-tier0-exception
  x-eventbus-tier0-exhausted-at
  x-eventbus-tier1-exception             (if applicable)
  x-eventbus-tier1-exhausted-at
  ...per-tier forensic trail
```

By the time a record reaches the DLT, its headers contain the complete forensic trail.

#### DLT Processing Strategies

| Strategy | Behavior |
|----------|----------|
| `STORE_ONLY` (default) | Record sits in DLT. External tooling or manual process handles reprocessing. |
| `ALERT_AND_STORE` | Publish to DLT + invoke a user-provided `AlertHandler` callback for PagerDuty, Slack, JIRA, etc. |
| `SCHEDULED_REPLAY` | A scheduled task reads DLT records older than a configurable threshold and republishes to the original topic. Bounded by batch size. Each replayed record gets `x-eventbus-replay-count`. If replay count exceeds max, record stays in DLT permanently. |
| `CUSTOM` | User provides a `DltRecordHandler` implementation that receives the full record + all metadata and decides what to do. |

#### DLT Admin Operations

| Operation | Description |
|-----------|-------------|
| `replayDlt(bindingName, count)` | Replay N records from DLT back to original topic |
| `replayDlt(bindingName, from, to)` | Replay records within a time window |
| `replayToTier(bindingName, tier, count)` | Replay DLT records to a specific retry tier |
| `purgeDlt(bindingName, olderThan)` | Delete DLT records older than a threshold |
| `dltStats(bindingName)` | Count, oldest record, newest record, breakdown by exception type |

---

### 4.8 Offset Management

| Concern | Strategy |
|---------|----------|
| `enable.auto.commit` | `false` — always, non-negotiable |
| Ack mode (single) | `MANUAL_IMMEDIATE` (default): `commitSync` after each record. `MANUAL`: Spring batches commits, committed on next poll. |
| Ack mode (batch) | `MANUAL`: explicit commit after full batch success, or selective commit on partial failure. |
| Partial batch failure | Commit offsets for all successfully processed records. Seek to first failure offset. No records skipped or duplicated. |
| Rebalance safety | `ConsumerAwareRebalanceListener` commits processed offsets on partition revocation. |
| Transactional (opt-in) | `KafkaTransactionManager` for exactly-once consume-transform-produce patterns. Per-binding opt-in. |

---

### 4.9 Rebalance Prevention

A multi-layered strategy to prevent rebalances caused by slow processing:

#### Layer 1 — Tuned Consumer Properties

Each binding exposes tunable properties with sensible defaults:

| Property | Default | Rationale |
|----------|---------|-----------|
| `max.poll.records` | 50 (single), 200 (batch) | Cap work per poll cycle |
| `max.poll.interval.ms` | 600000 (10 min) | Must exceed worst-case processing time |
| `session.timeout.ms` | 45000 | Tolerates GC pauses. ≥ 3 × heartbeat |
| `heartbeat.interval.ms` | 10000 | 1/3 of session timeout |

**Key relationship:** `max.poll.records × per-record-processing-time < max.poll.interval.ms` with comfortable margin.

#### Layer 2 — Processing Time Guard (Poll-Interval Safety Valve)

The listener adapter is time-aware. For batch listeners:

```
For each record[i]:
  Check: elapsed > 70% of max.poll.interval.ms?
  ├─ YES → STOP iterating
  │   • Commit offsets for records[0..i-1]
  │   • Seek partition to record[i] offset
  │   • Return → triggers next poll() immediately
  │   • Remaining records re-delivered in next poll
  └─ NO → Process normally
```

For single listeners, retry duration per record is capped:

```
retryBudget = min(
    configuredRetryDuration,
    maxPollIntervalMs × 0.7 − elapsedSinceLastPoll
)
```

If retries would exceed the budget, the record is DLQ'd immediately rather than risking a rebalance.

#### Layer 3 — Retry Budget Validation at Startup

```
totalRetryTime = maxAttempts × backoff (accounting for exponential growth)

Rule: totalRetryTime × maxPollRecords < maxPollIntervalMs × 0.7

Violation → fail fast with clear error explaining the incompatibility
```

#### Layer 4 — Circuit Breaker as Rebalance Prevention

When a downstream dependency degrades, retries pile up and processing slows. The circuit breaker detects this pattern early:

```
Dependency slows → failure rate threshold crossed
  → Circuit OPENS → immediate fast-fail
  → Lifecycle manager PAUSES container
  → Consumer stays in group, heartbeats continue, no records consumed
  → No rebalance. No message loss.
  → Circuit → HALF_OPEN → RESUME container
  → Circuit → CLOSED → normal processing
```

#### Layer 5 — Rebalance Listener Safety Net

Despite all preventive measures, rebalances happen (scaling, deployments, partition reassignment).

**On partition revoked:**
- `commitSync` for all fully processed records.
- Cancel in-flight retries for revoked partitions (per-partition cancellation flag).
- Log revoked partitions and committed offsets.

**On partition assigned:**
- Reset internal state (timers, batch progress) for newly assigned partitions.
- Processing starts from last committed offset.

#### Layer 6 — Static Group Membership (Optional)

Configure `group.instance.id` per consumer instance. When a consumer disconnects briefly (restart, deploy), the broker holds its partition assignment for `session.timeout.ms` rather than immediately rebalancing.

Auto-generated as: `{bindingName}-{hostname}-{containerIndex}`.

---

### 4.10 Deserialization Failure Handling

If a record can't be deserialized, it never reaches the listener — it fails inside `poll()`. Unhandled, this blocks the partition forever.

**Strategy:** Wrap actual key/value deserializers with `ErrorHandlingDeserializer`. On failure, it produces a record with a `null` payload and the exception stashed in headers (`ErrorHandlingDeserializer.VALUE_DESERIALIZER_EXCEPTION_HEADER`).

The listener adapter's first step checks for this header. If present:
1. Publish directly to DLT with deserialization error metadata.
2. Acknowledge the record.
3. Move on — no user handler invocation.

---

### 4.11 Graceful Shutdown

Hooks into Spring's `SmartLifecycle` with defined phase ordering.

**Shutdown sequence:**

```
SIGTERM / application shutdown
       │
       ▼
Phase 1: Stop accepting new registrations via admin API
       │
       ▼
Phase 2: Pause all containers (consumers stop fetching, stay in group)
       │
       ▼
Phase 3: Wait for in-flight processing to complete (configurable timeout, default 30s)
       │
       ▼
Phase 4: Commit final offsets for all completed records
       │
       ▼
Phase 5: Stop and destroy all containers (consumers leave groups)
```

**Phase ordering:** The connector shuts down before the `KafkaTemplate` (needed for DLQ publishes during draining) and before connection pools. Incorrect phase ordering would cause DLQ publishes during shutdown to fail silently.

The connector implements `SmartLifecycle` with a low phase number (high shutdown priority) and a `stop(Runnable callback)` method that orchestrates the sequence above.

---

### 4.12 Header Propagation & Enrichment

#### Inbound Enrichment (listener adapter, before user handler)

| Header | Value |
|--------|-------|
| `x-eventbus-binding-name` | Binding name |
| `x-eventbus-received-timestamp` | Time the record was received by the adapter |
| `x-eventbus-consumer-instance` | Hostname + container index |

#### DLT Enrichment (on publish to DLT)

Full forensic trail — see [Section 4.7](#47-dead-letter-topic-dlt-handling).

#### Retry Tier Enrichment (on publish to retry topic)

| Header | Value |
|--------|-------|
| `x-eventbus-original-topic` | Original topic |
| `x-eventbus-original-partition` | Original partition |
| `x-eventbus-original-offset` | Original offset |
| `x-eventbus-original-timestamp` | Original record timestamp |
| `x-eventbus-retry-tier` | Current tier number |
| `x-eventbus-retry-attempt` | Attempt within current tier |
| `x-eventbus-retry-timestamp` | Time the record was published to this tier |
| `x-eventbus-tier{N}-exception` | Exception class from tier N |
| `x-eventbus-tier{N}-exhausted-at` | Timestamp tier N was exhausted |

#### Trace Context Propagation

If the producing application set distributed tracing headers (W3C `traceparent`, OpenTelemetry), the connector propagates them through to the user handler and into DLT/retry records. This integrates with Micrometer Tracing without the connector owning the tracing implementation.

---

### 4.13 Observability Layer

#### Micrometer Metrics (per binding)

| Metric | Type | Tags |
|--------|------|------|
| `eventbus.records.processed` | Counter | binding, topic, partition |
| `eventbus.records.failed` | Counter | binding, topic, exception |
| `eventbus.records.dlq` | Counter | binding, topic, reason |
| `eventbus.retry.count` | Histogram | binding, tier |
| `eventbus.processing.duration` | Timer | binding, listenerType |
| `eventbus.batch.size` | Histogram | binding |
| `eventbus.circuit.state` | Gauge | binding (0=closed, 1=open, 2=half-open) |
| `eventbus.consumer.lag` | Gauge | binding, partition |
| `eventbus.container.state` | Gauge | binding (encoded as int) |
| `eventbus.dlt.depth` | Gauge | binding |
| `eventbus.retry.tier.depth` | Gauge | binding, tier |
| `eventbus.poll.interval.safety.triggered` | Counter | binding |

#### Structured Logging

Every log line during record processing includes MDC context:

```
bindingName, topic, partition, offset, consumerGroup, consumerInstance, retryTier
```

Example: `INFO [orders|orders.events|3|14209|order-group|host-0] Processing record`.

#### Health Indicator

A Spring Boot `HealthIndicator` that reports:

- `UP`: All registered containers are in their expected state and no circuit breakers are open.
- `DEGRADED`: Some circuit breakers are open or containers are paused (custom status).
- `DOWN`: Any container is in an unexpected state or the DLT publisher is unhealthy.

---

### 4.14 Admin API

An `EventBusAdmin` component exposed as a Spring bean and optionally via a REST controller.

**Container management:**

| Endpoint | Description |
|----------|-------------|
| `POST /eventbus/bindings` | Register a new binding dynamically (accepts `TopicBindingProperties` JSON) |
| `DELETE /eventbus/bindings/{name}` | Destroy and unregister |
| `POST /eventbus/bindings/{name}/start` | Start |
| `POST /eventbus/bindings/{name}/stop` | Stop |
| `POST /eventbus/bindings/{name}/pause` | Pause |
| `POST /eventbus/bindings/{name}/resume` | Resume |
| `GET /eventbus/bindings` | List all bindings with state |
| `GET /eventbus/bindings/{name}` | Binding detail with metrics summary |

**DLT operations:**

| Endpoint | Description |
|----------|-------------|
| `POST /eventbus/bindings/{name}/dlt/replay?count=N` | Replay N DLT records |
| `POST /eventbus/bindings/{name}/dlt/replay?from=...&to=...` | Replay by time window |
| `POST /eventbus/bindings/{name}/dlt/replay-to-tier?tier=N&count=M` | Replay to specific tier |
| `GET /eventbus/bindings/{name}/dlt/stats` | DLT statistics |
| `DELETE /eventbus/bindings/{name}/dlt?olderThan=...` | Purge old DLT records |

---

## 5. Key Interfaces

```java
/**
 * Core event bus connector — entry point for all operations.
 */
public interface EventBusConnector {

    void register(TopicBindingProperties binding);
    void unregister(String bindingName);

    void start(String bindingName);
    void stop(String bindingName);
    void pause(String bindingName);
    void resume(String bindingName);
    void destroy(String bindingName);

    ContainerState getState(String bindingName);
    List<ContainerRegistration> listBindings();

    Health health();
}

/**
 * User implements this for single-record processing.
 * The connector handles everything else.
 */
@FunctionalInterface
public interface RecordHandler<K, V> {
    void handle(ConsumerRecord<K, V> record);
}

/**
 * User implements this for batch processing.
 * Records arrive in partition-offset order.
 */
@FunctionalInterface
public interface BatchRecordHandler<K, V> {
    void handle(ConsumerRecord<K, V> record);
    // NOTE: receives one record at a time from the batch.
    // The connector iterates and manages partial failure.
}

/**
 * Optional — invoked when DLT strategy is ALERT_AND_STORE.
 */
@FunctionalInterface
public interface DltAlertHandler {
    void onDeadLetter(DltRecord record);
}

/**
 * Optional — invoked when DLT strategy is CUSTOM.
 */
@FunctionalInterface
public interface DltRecordHandler {
    void handle(DltRecord record);
}

/**
 * Exception classifier — determines routing for failed records.
 */
@FunctionalInterface
public interface ExceptionClassifier {
    ExceptionRoutingDecision classify(Throwable exception, int currentTier);
}

/**
 * Routing decision from the exception classifier.
 */
public record ExceptionRoutingDecision(
    ExceptionRouting routing,
    int targetTier        // used when routing is SKIP_TO_TIER
) {
    public static ExceptionRoutingDecision nextTier() {
        return new ExceptionRoutingDecision(ExceptionRouting.NEXT_TIER, -1);
    }

    public static ExceptionRoutingDecision deadLetter() {
        return new ExceptionRoutingDecision(ExceptionRouting.DEAD_LETTER, -1);
    }

    public static ExceptionRoutingDecision skipToTier(int tier) {
        return new ExceptionRoutingDecision(ExceptionRouting.SKIP_TO_TIER, tier);
    }
}

/**
 * DLT record with full forensic context.
 */
public record DltRecord(
    String originalTopic,
    int originalPartition,
    long originalOffset,
    long originalTimestamp,
    byte[] key,
    byte[] value,
    Map<String, byte[]> headers,
    String dltReason,
    int totalAttempts,
    long firstFailureTimestamp,
    String lastExceptionClass,
    String lastExceptionMessage,
    List<TierFailureRecord> tierHistory
) {}

/**
 * Per-tier failure information.
 */
public record TierFailureRecord(
    int tier,
    String exceptionClass,
    long exhaustedAt,
    int attempts
) {}
```

---

## 6. Threading Model

```
ConcurrentMessageListenerContainer (concurrency = 3)
  ├── KafkaMessageListenerContainer-0  →  Thread-0  →  Partitions [0, 1]
  ├── KafkaMessageListenerContainer-1  →  Thread-1  →  Partitions [2, 3]
  └── KafkaMessageListenerContainer-2  →  Thread-2  →  Partitions [4, 5]
```

Each consumer thread gets a disjoint set of partitions. No two threads ever process the same partition.

**Shared vs. per-thread components:**

| Component | Scope | Rationale |
|-----------|-------|-----------|
| `CircuitBreaker` | One per binding, shared | Protects a shared downstream dependency — all threads must see the open circuit |
| `Retry` | Config per binding, context per invocation | Stateless per call, no shared mutable state |
| `ExceptionClassifier` | One per binding, shared | Stateless, read-only configuration |
| `DLT KafkaTemplate` | One per binding, shared | Thread-safe, connection-pooled |
| `Offset tracking map` | Per consumer thread | No cross-thread offset interference |

---

## 7. Configuration Validation

Performed at startup and at dynamic registration time. Fails fast with descriptive error messages.

| Validation | Rule |
|-----------|------|
| Retry budget vs poll interval | `totalRetryTime × maxPollRecords < maxPollIntervalMs × 0.7` |
| Session timeout vs heartbeat | `sessionTimeoutMs ≥ 3 × heartbeatIntervalMs` |
| Concurrency vs partition count | Warn if `concurrency > partitionCount` (wasted threads) |
| DLT topic existence | Optionally verify DLT topic exists at startup |
| Ack mode + listener type | Batch listener requires MANUAL or MANUAL_IMMEDIATE; reject incompatible modes |
| CircuitBreaker + lifecycle | If circuit breaker is enabled with auto-pause, lifecycle manager must be wired |
| Tier configuration | Delays must be strictly increasing across tiers |
| Non-retryable exceptions | Cannot overlap with retryable exceptions list |
| Skip-to-tier mapping | Target tier must exist in the tier list |
| Strict ordering + tiered retry | STRICT mode is incompatible with topic-based tiers — fail if both configured |

---

## 8. Idempotency Support

The connector guarantees at-least-once delivery. For effective exactly-once, the user handler must be idempotent. The connector assists in two ways:

1. **Natural deduplication key:** The `topic-partition-offset` triple is available in every `ConsumerRecord` and can be used as a deduplication key by the user handler.

2. **Optional idempotency interceptor (opt-in per binding):** An interceptor that checks a fast store (Redis, in-memory Bloom filter, or database table) before invoking the handler. If the offset has already been processed, the interceptor skips the handler and acknowledges the record.

The interceptor follows the `RecordHandler` decorator pattern — it wraps the user handler transparently.

---

## 9. Multi-Cluster Support

For deployments consuming from multiple Kafka clusters (primary and DR):

- The container factory accepts an explicit `ConsumerFactory` rather than using a single autowired instance.
- The admin API for dynamic registration accepts a `cluster` alias, which maps to a pre-configured `ConsumerFactory`.
- The `TopicBindingProperties` gains an optional `cluster` field (default: `"default"`).
- The registry key becomes `{cluster}:{bindingName}`.
- Each cluster's `ConsumerFactory` and `KafkaTemplate` (for DLT) are configured independently.

---

## 10. Testing Strategy

The connector ships with a test harness module:

| Component | Purpose |
|-----------|---------|
| `EmbeddedKafkaBroker` integration test base | Auto-configures the connector with test bindings |
| `TestRecordHandler` | Captures received records, supports latches for async assertions |
| `DltAssertions` | Reads from DLT topic, asserts on record content, headers, and count |
| `ContainerStateAssertions` | Verifies lifecycle transitions (start → pause → resume → stop) |
| `RetryTierAssertions` | Verifies records flow through tiers correctly |
| `FailingHandler` | Configurable handler that fails N times then succeeds — for retry testing |
| `SlowHandler` | Configurable handler with artificial delay — for poll-interval safety testing |

---

## 11. Package Structure

```
eventbus-connector/
├── build.gradle
├── settings.gradle
└── src/
    ├── main/java/com/enterprise/eventbus/
    │   ├── EventBusConnector.java              # Core interface
    │   ├── EventBusConnectorAutoConfiguration  # Spring Boot auto-config
    │   │
    │   ├── model/
    │   │   ├── ListenerType.java               # SINGLE, BATCH
    │   │   ├── ContainerState.java             # CREATED, RUNNING, PAUSED, STOPPED, DESTROYED
    │   │   ├── DltStrategy.java                # STORE_ONLY, ALERT_AND_STORE, SCHEDULED_REPLAY, CUSTOM
    │   │   ├── BatchFailureStrategy.java       # SEEK_TO_FAILED, DLQ_AND_CONTINUE
    │   │   ├── OrderingMode.java               # RELAXED_IN_RETRY, STRICT
    │   │   ├── ExceptionRouting.java           # NEXT_TIER, SKIP_TO_TIER, DEAD_LETTER, PAUSE_CONTAINER
    │   │   └── ExceptionRoutingDecision.java   # Record: routing + targetTier
    │   │
    │   ├── config/
    │   │   ├── EventBusProperties.java         # Root @ConfigurationProperties
    │   │   ├── TopicBindingProperties.java     # Per-binding configuration
    │   │   ├── RetryProperties.java            # Retry tiers configuration
    │   │   ├── RetryTierProperties.java        # Single tier config
    │   │   ├── CircuitBreakerProperties.java   # CircuitBreaker per-binding config
    │   │   ├── DltProperties.java              # DLT strategy and replay config
    │   │   └── ConfigurationValidator.java     # Startup coherence validation
    │   │
    │   ├── handler/
    │   │   ├── RecordHandler.java              # User-facing single record handler
    │   │   ├── BatchRecordHandler.java         # User-facing batch handler
    │   │   ├── DltAlertHandler.java            # Alert callback for DLT
    │   │   ├── DltRecordHandler.java           # Custom DLT processing
    │   │   └── ExceptionClassifier.java        # Exception → routing decision
    │   │
    │   ├── container/
    │   │   ├── ContainerFactory.java           # Builds containers from config
    │   │   ├── ContainerRegistry.java          # ConcurrentHashMap registry
    │   │   ├── ContainerRegistration.java      # Record: container + metadata
    │   │   └── LifecycleManager.java           # Start/stop/pause/resume/destroy
    │   │
    │   ├── listener/
    │   │   ├── SingleRecordListenerAdapter.java     # AcknowledgingMessageListener
    │   │   ├── BatchRecordListenerAdapter.java      # BatchAcknowledgingMessageListener
    │   │   ├── RetryTierListenerAdapter.java        # Delay-aware retry tier listener
    │   │   ├── DeserializationFailureHandler.java   # Poison pill detection
    │   │   └── PollIntervalSafetyValve.java         # Time-budget guard
    │   │
    │   ├── resilience/
    │   │   ├── ResilienceDecorator.java         # Composes Retry + CircuitBreaker
    │   │   ├── RetryConfigFactory.java          # Builds Resilience4j RetryConfig
    │   │   ├── CircuitBreakerConfigFactory.java # Builds CircuitBreakerConfig
    │   │   ├── CircuitBreakerLifecycleBridge.java  # CB state → pause/resume
    │   │   └── RetryBudgetCalculator.java       # Validates retry budget vs poll interval
    │   │
    │   ├── offset/
    │   │   ├── PartitionOffsetManager.java      # Tracks per-partition offsets for batch
    │   │   ├── EventBusRebalanceListener.java   # ConsumerAwareRebalanceListener
    │   │   └── BatchOffsetCommitStrategy.java   # Partial commit logic
    │   │
    │   ├── dlq/
    │   │   ├── DltPublisher.java                # Synchronous DLT publishing
    │   │   ├── DltRecord.java                   # Record: full forensic context
    │   │   ├── TierFailureRecord.java           # Record: per-tier failure info
    │   │   ├── DltHeaderEnricher.java           # Header enrichment for DLT records
    │   │   ├── RetryHeaderEnricher.java         # Header enrichment for retry records
    │   │   ├── DltReplayService.java            # Scheduled/on-demand DLT replay
    │   │   └── DltStatsService.java             # DLT statistics
    │   │
    │   ├── observability/
    │   │   ├── EventBusMetrics.java             # Micrometer metric registration
    │   │   ├── EventBusHealthIndicator.java     # Spring Boot HealthIndicator
    │   │   └── MdcContextPropagator.java        # Structured logging MDC
    │   │
    │   ├── admin/
    │   │   ├── EventBusAdmin.java               # Programmatic admin API
    │   │   └── EventBusAdminController.java     # Optional REST endpoints
    │   │
    │   └── exception/
    │       ├── EventBusException.java           # Base exception
    │       ├── InvalidContainerStateException.java
    │       ├── BindingAlreadyExistsException.java
    │       ├── BindingNotFoundException.java
    │       ├── ConfigurationValidationException.java
    │       ├── BatchProcessingException.java    # Wraps failed index for batch
    │       └── DltPublishException.java
    │
    ├── main/resources/
    │   ├── META-INF/spring/
    │   │   └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
    │   └── application-eventbus-defaults.yml    # Default property values
    │
    └── test/java/com/enterprise/eventbus/
        ├── test/
        │   ├── EmbeddedKafkaTestBase.java       # Integration test base
        │   ├── TestRecordHandler.java           # Captures records for assertions
        │   ├── FailingHandler.java              # Configurable failure handler
        │   ├── SlowHandler.java                 # Configurable slow handler
        │   ├── DltAssertions.java               # DLT verification utilities
        │   ├── ContainerStateAssertions.java    # Lifecycle assertions
        │   └── RetryTierAssertions.java         # Tier flow assertions
        │
        ├── SingleRecordListenerAdapterTest.java
        ├── BatchRecordListenerAdapterTest.java
        ├── ContainerFactoryTest.java
        ├── LifecycleManagerTest.java
        ├── TieredRetryIntegrationTest.java
        ├── PartialBatchFailureTest.java
        ├── CircuitBreakerIntegrationTest.java
        ├── RebalanceHandlingTest.java
        ├── DeserializationFailureTest.java
        ├── GracefulShutdownTest.java
        ├── DltReplayTest.java
        ├── PollIntervalSafetyValveTest.java
        └── ConfigurationValidationTest.java
```

---

## 12. Technology Matrix

| Concern | Technology | Rationale |
|---------|-----------|-----------|
| Listener type | `AcknowledgingMessageListener` / `BatchAcknowledgingMessageListener` | Explicit offset control |
| Container | `ConcurrentMessageListenerContainer` | Partition-based concurrency |
| In-memory retry | Resilience4j `Retry` | Richer than `@Retryable`, composable, backoff strategies |
| Circuit breaking | Resilience4j `CircuitBreaker` | State-change events integrate with pause/resume |
| DLQ publishing | `KafkaTemplate` with idempotent producer | Zero-loss synchronous publish |
| Offset commit | Manual via `Acknowledgment` + `commitSync` on partial failure | Fine-grained partition-level control |
| Configuration | `@ConfigurationProperties` with records | Type-safe, immutable, validated |
| Dynamic containers | Factory + Registry pattern | Runtime topic subscription |
| Deserialization safety | `ErrorHandlingDeserializer` | Prevents partition-blocking poison pills |
| Metrics | Micrometer | Spring Boot native, Prometheus/Grafana compatible |
| Health | Spring Boot `HealthIndicator` | Actuator integration |
| Build | Gradle 8.x | Incremental builds, dependency management |
| Java | 21 (LTS) | Records, sealed interfaces, pattern matching, virtual threads awareness |

---

## Appendix: Configuration Reference

### Sample `application.yml`

```yaml
eventbus:
  bindings:
    orders:
      topic: orders.events
      group-id: order-processing-group
      listener-type: BATCH
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 3
      max-poll-records: 200
      max-poll-interval-ms: 600000
      session-timeout-ms: 45000
      heartbeat-interval-ms: 10000
      auto-startup: true
      static-membership: true
      batch-failure-strategy: SEEK_TO_FAILED
      ordering-mode: RELAXED_IN_RETRY

      retry:
        tier0:
          max-attempts: 3
          initial-backoff-ms: 100
          multiplier: 2.0
          max-backoff-ms: 2000
        tiers:
          - delay-ms: 10000
            max-attempts: 3
            topic-suffix: retry-1
            concurrency: 1
          - delay-ms: 60000
            max-attempts: 3
            topic-suffix: retry-2
            concurrency: 1
          - delay-ms: 300000
            max-attempts: 3
            topic-suffix: retry-3
            concurrency: 1
        retryable-exceptions:
          - java.net.SocketTimeoutException
          - org.springframework.dao.OptimisticLockingFailureException
          - org.springframework.dao.TransientDataAccessException
        non-retryable-exceptions:
          - com.enterprise.schema.SchemaValidationException
          - com.enterprise.domain.BusinessRuleViolationException
        skip-to-tier-mapping:
          javax.ws.rs.ServiceUnavailableException: 2

      circuit-breaker:
        enabled: true
        failure-rate-threshold: 50.0
        slow-call-rate-threshold: 80.0
        slow-call-duration-ms: 5000
        wait-duration-in-open-state-ms: 60000
        sliding-window-size: 100
        minimum-number-of-calls: 10
        permitted-number-of-calls-in-half-open-state: 5
        automatic-pause-on-open: true

      dlt:
        topic-suffix: DLT
        strategy: ALERT_AND_STORE
        replay:
          max-replays: 3
          cooldown-ms: 3600000
          batch-size: 100

    payments:
      topic: payments.events
      group-id: payment-processing-group
      listener-type: SINGLE
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 2
      ordering-mode: STRICT

      retry:
        tier0:
          max-attempts: 5
          initial-backoff-ms: 200
          multiplier: 2.0
        # No topic-based tiers — strict ordering mode

      circuit-breaker:
        enabled: true
        failure-rate-threshold: 30.0
        automatic-pause-on-open: true

      dlt:
        strategy: STORE_ONLY
```

### Minimal Configuration (Sensible Defaults)

```yaml
eventbus:
  bindings:
    notifications:
      topic: notification.events
      group-id: notification-group
```

This uses all defaults: single listener, manual-immediate ack, concurrency 1, 3 retry tiers with standard delays, store-only DLT, no circuit breaker.

---

*End of Technical Design Document*
