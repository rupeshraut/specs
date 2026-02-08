# 🍃 MongoDB with Spring Data Cheat Sheet

> **Purpose:** Production patterns for MongoDB in Spring Boot — queries, aggregations, indexing, transactions, and performance. Reference before designing any repository or data model.
> **Stack context:** Java 21+ / Spring Boot 3.x / Spring Data MongoDB / MongoDB 7.x

---

## 📋 The MongoDB Design Decision Framework

Before creating any collection, answer:

| Question | Determines |
|----------|-----------|
| What are the **access patterns**? (read-heavy, write-heavy, both) | Schema design |
| Will you query by **which fields**? | Index strategy |
| How often does the data **change**? | Embed vs reference |
| How **large** can a document grow? | 16MB limit, bucket pattern |
| Is **atomicity** needed across documents? | Transactions |
| Do you need **temporal queries**? | TTL indexes, versioning |
| What's the **cardinality** of relationships? | 1:1, 1:few, 1:many, many:many |

---

## 🏗️ Pattern 1: Document Modeling

### Embedding vs Referencing Decision Tree

```
Is the related data...
├── Always accessed together?
│   └── YES → EMBED
│       (e.g., order + line items, payment + fee breakdown)
│
├── Updated independently and frequently?
│   └── YES → REFERENCE
│       (e.g., customer profile updated separately from payments)
│
├── Growing unboundedly?
│   └── YES → REFERENCE (or bucket pattern)
│       (e.g., audit log entries, transaction history)
│
├── Small and rarely changes?
│   └── YES → EMBED
│       (e.g., address in an order, shipping details)
│
└── Shared across many parent documents?
    └── YES → REFERENCE
        (e.g., product catalog referenced by many orders)
```

### Document Design with Records

```java
// ── Payment aggregate — embeds value objects, references entities ──
@Document("payments")
@CompoundIndex(name = "customer_status_idx", def = "{'customerId': 1, 'status': 1}")
@CompoundIndex(name = "created_status_idx", def = "{'createdAt': -1, 'status': 1}")
@CompoundIndex(name = "idempotency_idx", def = "{'idempotencyKey': 1}", unique = true)
public record Payment(
    @Id String id,
    String customerId,              // Reference to customer collection
    String idempotencyKey,
    PaymentStatus status,
    Money amount,                   // Embedded value object
    FeeBreakdown fees,              // Embedded value object
    PaymentMethod method,           // Embedded value object
    List<StatusChange> statusHistory, // Embedded audit trail (bounded)
    GatewayResponse gatewayResponse,  // Embedded
    Instant createdAt,
    Instant updatedAt,
    @Version Long version           // Optimistic locking
) {
    // ── Embedded value objects ──
    public record Money(BigDecimal value, String currency) {}
    public record FeeBreakdown(BigDecimal processingFee, BigDecimal platformFee, BigDecimal total) {}
    public record PaymentMethod(String type, String last4, String brand) {}
    public record StatusChange(PaymentStatus from, PaymentStatus to, String reason, Instant changedAt) {}
    public record GatewayResponse(String transactionId, String authCode, String responseCode) {}
}

// ── Customer — separate collection (updated independently) ──
@Document("customers")
public record Customer(
    @Id String id,
    String name,
    String email,
    Address defaultAddress,         // Embedded (1:1, changes rarely)
    List<String> paymentMethodIds,  // References to payment_methods collection
    Instant createdAt
) {
    public record Address(String line1, String line2, String city, String state, String zip) {}
}
```

### Bounded List Pattern (Prevent Unbounded Growth)

```java
// ❌ Unbounded array — will hit 16MB limit
@Document("payments")
public record Payment(
    @Id String id,
    List<AuditEntry> auditLog  // Could grow to millions of entries!
) {}

// ✅ Bounded in document + overflow to separate collection
@Document("payments")
public record Payment(
    @Id String id,
    List<StatusChange> recentStatusChanges,  // Last 10 only (bounded)
    Instant createdAt
) {}

@Document("payment_audit_log")    // Overflow collection
@CompoundIndex(name = "payment_time_idx", def = "{'paymentId': 1, 'timestamp': -1}")
public record PaymentAuditEntry(
    @Id String id,
    String paymentId,             // Reference back to payment
    String action,
    String details,
    Instant timestamp
) {}
```

---

## 🔍 Pattern 2: Repository Patterns

### Spring Data Repository — Query Methods

```java
public interface PaymentRepository extends MongoRepository<Payment, String> {

    // ── Derived queries (Spring generates the query) ──
    List<Payment> findByCustomerIdAndStatus(String customerId, PaymentStatus status);
    Optional<Payment> findByIdempotencyKey(String idempotencyKey);
    Page<Payment> findByStatusIn(List<PaymentStatus> statuses, Pageable pageable);
    long countByStatusAndCreatedAtAfter(PaymentStatus status, Instant after);
    boolean existsByIdempotencyKey(String key);

    // ── @Query with MongoDB JSON ──
    @Query("{ 'customerId': ?0, 'status': { $in: ?1 }, 'createdAt': { $gte: ?2 } }")
    List<Payment> findByCustomerAndStatusSince(
        String customerId, List<PaymentStatus> statuses, Instant since);

    // ── Projection — return subset of fields ──
    @Query(value = "{ 'customerId': ?0 }", fields = "{ 'id': 1, 'amount': 1, 'status': 1 }")
    List<PaymentSummary> findSummariesByCustomerId(String customerId);

    // ── Sort and limit ──
    List<Payment> findTop10ByStatusOrderByCreatedAtDesc(PaymentStatus status);

    // ── Exists check (efficient — doesn't load document) ──
    @Query(value = "{ 'customerId': ?0, 'status': 'COMPLETED' }", exists = true)
    boolean hasCompletedPayments(String customerId);

    // ── Delete ──
    long deleteByStatusAndCreatedAtBefore(PaymentStatus status, Instant before);
}

// Projection interface (only loads specified fields)
public interface PaymentSummary {
    String getId();
    Payment.Money getAmount();
    PaymentStatus getStatus();
}
```

### MongoTemplate — Complex Queries

```java
@Repository
public class PaymentCustomRepository {

    private final MongoTemplate mongoTemplate;

    // ── Dynamic query building ──
    public Page<Payment> search(PaymentSearchCriteria criteria, Pageable pageable) {
        var query = new Query();

        if (criteria.customerId() != null) {
            query.addCriteria(Criteria.where("customerId").is(criteria.customerId()));
        }
        if (criteria.statuses() != null && !criteria.statuses().isEmpty()) {
            query.addCriteria(Criteria.where("status").in(criteria.statuses()));
        }
        if (criteria.fromDate() != null) {
            query.addCriteria(Criteria.where("createdAt").gte(criteria.fromDate()));
        }
        if (criteria.toDate() != null) {
            query.addCriteria(Criteria.where("createdAt").lte(criteria.toDate()));
        }
        if (criteria.minAmount() != null) {
            query.addCriteria(Criteria.where("amount.value").gte(criteria.minAmount()));
        }

        long total = mongoTemplate.count(query, Payment.class);
        query.with(pageable);
        var payments = mongoTemplate.find(query, Payment.class);

        return new PageImpl<>(payments, pageable, total);
    }

    // ── Atomic field update (no full document read-write) ──
    public boolean updateStatus(String paymentId, PaymentStatus from, PaymentStatus to, String reason) {
        var result = mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is(paymentId)
                    .and("status").is(from)),    // Conditional — prevents invalid transitions
            new Update()
                .set("status", to)
                .set("updatedAt", Instant.now())
                .push("recentStatusChanges")
                    .atPosition(Position.FIRST)
                    .each(new Payment.StatusChange(from, to, reason, Instant.now()))
                .currentDate("updatedAt"),
            Payment.class
        );
        return result.getModifiedCount() > 0;
    }

    // ── Atomic increment ──
    public void incrementRetryCount(String paymentId) {
        mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is(paymentId)),
            new Update().inc("retryCount", 1).set("lastRetryAt", Instant.now()),
            Payment.class
        );
    }

    // ── Find and modify atomically ──
    public Payment claimForProcessing(String paymentId) {
        return mongoTemplate.findAndModify(
            Query.query(Criteria.where("_id").is(paymentId)
                    .and("status").is(PaymentStatus.PENDING)),
            new Update()
                .set("status", PaymentStatus.PROCESSING)
                .set("processingStartedAt", Instant.now())
                .set("processorInstance", hostname()),
            FindAndModifyOptions.options().returnNew(true),
            Payment.class
        );
    }

    // ── Bulk write (high performance) ──
    public void bulkUpdateStatuses(Map<String, PaymentStatus> updates) {
        var ops = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, Payment.class);
        updates.forEach((id, status) ->
            ops.updateOne(
                Query.query(Criteria.where("_id").is(id)),
                new Update().set("status", status).set("updatedAt", Instant.now())
            ));
        var result = ops.execute();
        log.info("Bulk update: {} modified of {} requested", result.getModifiedCount(), updates.size());
    }
}
```

---

## 📊 Pattern 3: Aggregation Pipeline

### Common Aggregation Patterns

```java
@Repository
public class PaymentAnalyticsRepository {

    private final MongoTemplate mongoTemplate;

    // ── Group by status with totals ──
    public List<StatusSummary> aggregateByStatus(Instant from, Instant to) {
        var aggregation = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("createdAt").gte(from).lte(to)),
            Aggregation.group("status")
                .count().as("count")
                .sum("amount.value").as("totalAmount")
                .avg("amount.value").as("avgAmount")
                .min("createdAt").as("earliest")
                .max("createdAt").as("latest"),
            Aggregation.sort(Sort.Direction.DESC, "totalAmount")
        );

        return mongoTemplate.aggregate(aggregation, "payments", StatusSummary.class)
            .getMappedResults();
    }

    public record StatusSummary(
        @Field("_id") PaymentStatus status,
        long count,
        BigDecimal totalAmount,
        BigDecimal avgAmount,
        Instant earliest,
        Instant latest
    ) {}

    // ── Daily revenue report ──
    public List<DailyRevenue> dailyRevenue(Instant from, Instant to) {
        var aggregation = Aggregation.newAggregation(
            Aggregation.match(
                Criteria.where("status").is("CAPTURED")
                    .and("createdAt").gte(from).lte(to)),
            Aggregation.project()
                .and("createdAt").dateAsFormattedString("%Y-%m-%d").as("date")
                .and("amount.value").as("amount")
                .and("amount.currency").as("currency")
                .and("fees.total").as("fee"),
            Aggregation.group("date", "currency")
                .sum("amount").as("totalRevenue")
                .sum("fee").as("totalFees")
                .count().as("transactionCount"),
            Aggregation.sort(Sort.Direction.ASC, "_id.date")
        );

        return mongoTemplate.aggregate(aggregation, "payments", DailyRevenue.class)
            .getMappedResults();
    }

    // ── Top customers by spend ──
    public List<TopCustomer> topCustomers(int limit, Instant since) {
        var aggregation = Aggregation.newAggregation(
            Aggregation.match(
                Criteria.where("status").is("CAPTURED")
                    .and("createdAt").gte(since)),
            Aggregation.group("customerId")
                .sum("amount.value").as("totalSpend")
                .count().as("paymentCount")
                .avg("amount.value").as("avgPayment"),
            Aggregation.sort(Sort.Direction.DESC, "totalSpend"),
            Aggregation.limit(limit),
            // Lookup customer details
            Aggregation.lookup("customers", "_id", "_id", "customerDetails"),
            Aggregation.unwind("customerDetails", true)
        );

        return mongoTemplate.aggregate(aggregation, "payments", TopCustomer.class)
            .getMappedResults();
    }

    // ── Bucket analysis — payment amount distribution ──
    public List<AmountBucket> paymentDistribution() {
        var aggregation = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("status").is("CAPTURED")),
            Aggregation.bucket("amount.value")
                .withBoundaries(0, 10, 50, 100, 500, 1000, 5000, 50000)
                .withDefaultBucket("50000+")
                .andOutputCount().as("count")
                .andOutput("amount.value").sum().as("totalAmount")
        );

        return mongoTemplate.aggregate(aggregation, "payments", AmountBucket.class)
            .getMappedResults();
    }

    // ── Windowed aggregation — hourly throughput ──
    public List<HourlyThroughput> hourlyThroughput(Instant date) {
        var aggregation = Aggregation.newAggregation(
            Aggregation.match(
                Criteria.where("createdAt")
                    .gte(date).lt(date.plus(Duration.ofDays(1)))),
            Aggregation.project()
                .and("createdAt").hour().as("hour")
                .and("status").as("status"),
            Aggregation.group("hour", "status").count().as("count"),
            Aggregation.sort(Sort.Direction.ASC, "_id.hour")
        );

        return mongoTemplate.aggregate(aggregation, "payments", HourlyThroughput.class)
            .getMappedResults();
    }
}
```

---

## 📇 Pattern 4: Indexing Strategy

### Index Decision Guide

| Query Pattern | Index Type | Example |
|--------------|-----------|---------|
| Equality on single field | **Single field** | `{ customerId: 1 }` |
| Equality + sort | **Compound** | `{ customerId: 1, createdAt: -1 }` |
| Equality + range | **Compound** (equality first) | `{ status: 1, createdAt: -1 }` |
| Text search | **Text index** | `{ description: "text" }` |
| Unique constraint | **Unique** | `{ idempotencyKey: 1 }, unique` |
| Auto-expiry | **TTL** | `{ expiresAt: 1 }, expireAfterSeconds: 0` |
| Partial (conditional) | **Partial** | `{ status: 1 } where status = 'PENDING'` |
| Array field queries | **Multikey** | `{ "tags": 1 }` |
| Geospatial | **2dsphere** | `{ location: "2dsphere" }` |

### ESR Rule (Equality → Sort → Range)

```
When building compound indexes, order fields as:
  1. EQUALITY fields first     (status = "PENDING")
  2. SORT fields next          (ORDER BY createdAt DESC)
  3. RANGE fields last         (createdAt > '2026-01-01')

Example query:
  db.payments.find({
    status: "PENDING",           ← Equality
    createdAt: { $gte: cutoff }  ← Range
  }).sort({ createdAt: -1 })     ← Sort

Best index:
  { status: 1, createdAt: -1 }  ← Equality field, then sort/range field
```

### Index Configuration in Spring

```java
// ── Annotation-based (simple cases) ──
@Document("payments")
@CompoundIndex(name = "customer_status_idx", def = "{'customerId': 1, 'status': 1}")
@CompoundIndex(name = "status_created_idx", def = "{'status': 1, 'createdAt': -1}")
@CompoundIndex(name = "idempotency_idx", def = "{'idempotencyKey': 1}", unique = true)
public record Payment(...) {}

// ── Programmatic (complex cases, partial indexes) ──
@Configuration
public class MongoIndexConfig {

    @Bean
    public MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(List.of());
    }

    @EventListener(ApplicationReadyEvent.class)
    public void createIndexes(MongoTemplate mongoTemplate) {
        var indexOps = mongoTemplate.indexOps("payments");

        // Partial index — only index PENDING payments (smaller, faster)
        indexOps.ensureIndex(new Index()
            .on("status", Sort.Direction.ASC)
            .on("createdAt", Sort.Direction.ASC)
            .partial(PartialIndexFilter.of(Criteria.where("status").is("PENDING")))
            .named("pending_payments_idx"));

        // TTL index — auto-delete processed events after 24h
        mongoTemplate.indexOps("processed_events").ensureIndex(
            new Index()
                .on("processedAt", Sort.Direction.ASC)
                .expire(Duration.ofHours(24))
                .named("processed_events_ttl"));

        // Sparse index — only documents that HAVE the field
        indexOps.ensureIndex(new Index()
            .on("gatewayResponse.transactionId", Sort.Direction.ASC)
            .sparse()
            .named("gateway_txn_idx"));
    }
}
```

### Index Anti-Patterns

```
❌ Index every field individually
   → Too many indexes slow writes and consume memory

❌ Compound index in wrong order
   → { createdAt: -1, status: 1 } won't help filter by status

❌ Unused indexes
   → Check with db.collection.aggregate([{$indexStats:{}}])

❌ No index on foreign key fields
   → Lookups become collection scans

❌ Missing index for sort fields
   → In-memory sort on large collections = slow + OOM risk

✅ Use explain() to verify index usage
   → db.payments.find({status:"PENDING"}).explain("executionStats")
```

---

## 🔐 Pattern 5: Transactions

### When to Use MongoDB Transactions

```
✅ USE transactions when:
  • Updating multiple documents that MUST be consistent
  • Implementing outbox pattern (business doc + outbox in same tx)
  • Moving money between accounts (debit + credit atomically)
  • Saga step that modifies multiple collections

❌ DON'T USE transactions when:
  • Single document update (already atomic in MongoDB)
  • Read-only operations
  • Operations across different databases/shards (limited support)
  • High-throughput paths where single-doc atomicity suffices
```

### Transaction Patterns

```java
@Component
public class TransactionalPaymentService {

    private final MongoTemplate mongoTemplate;
    private final MongoTransactionManager transactionManager;

    // ── Pattern A: TransactionTemplate (programmatic) ──
    public Payment capturePayment(String paymentId, Money settledAmount) {
        var txTemplate = new TransactionTemplate(transactionManager);
        return txTemplate.execute(status -> {
            // All operations in this lambda are in ONE transaction

            // 1. Update payment
            var payment = mongoTemplate.findById(paymentId, Payment.class);
            if (payment.status() != PaymentStatus.AUTHORIZED) {
                throw new IllegalStateException("Cannot capture from " + payment.status());
            }
            var captured = payment.withStatus(PaymentStatus.CAPTURED)
                                  .withSettledAmount(settledAmount);
            mongoTemplate.save(captured);

            // 2. Write to outbox
            mongoTemplate.insert(OutboxEvent.forPaymentCaptured(captured));

            // 3. Update customer stats
            mongoTemplate.updateFirst(
                Query.query(Criteria.where("_id").is(payment.customerId())),
                new Update().inc("totalSpend", settledAmount.value().doubleValue()),
                Customer.class
            );

            return captured;
            // All 3 writes committed atomically OR all rolled back
        });
    }

    // ── Pattern B: @Transactional annotation ──
    @Transactional
    public TransferResult transfer(String fromAccount, String toAccount, BigDecimal amount) {
        // Debit source
        var debitResult = mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is(fromAccount)
                    .and("balance").gte(amount)),
            new Update().inc("balance", amount.negate().doubleValue()),
            Account.class
        );
        if (debitResult.getModifiedCount() == 0) {
            throw new InsufficientFundsException(fromAccount);
        }

        // Credit destination
        mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is(toAccount)),
            new Update().inc("balance", amount.doubleValue()),
            Account.class
        );

        return TransferResult.success(fromAccount, toAccount, amount);
    }
}

// Enable MongoDB transactions
@Configuration
public class MongoTransactionConfig {

    @Bean
    public MongoTransactionManager transactionManager(MongoDatabaseFactory dbFactory) {
        return new MongoTransactionManager(dbFactory);
    }
}
```

---

## 🔒 Pattern 6: Distributed Locking

```java
@Component
public class MongoDistributedLock {

    private final MongoTemplate mongoTemplate;

    /**
     * Try to acquire a lock. Returns true if acquired.
     * Lock auto-expires after ttl to prevent deadlocks.
     */
    public boolean tryAcquire(String lockId, String owner, Duration ttl) {
        try {
            mongoTemplate.insert(new LockDocument(
                lockId, owner, Instant.now(), Instant.now().plus(ttl)));
            return true;
        } catch (DuplicateKeyException e) {
            // Lock exists — check if expired
            var existing = mongoTemplate.findById(lockId, LockDocument.class);
            if (existing != null && existing.expiresAt().isBefore(Instant.now())) {
                // Expired — try to steal it
                var result = mongoTemplate.updateFirst(
                    Query.query(Criteria.where("_id").is(lockId)
                            .and("expiresAt").lt(Instant.now())),
                    Update.update("owner", owner)
                          .set("acquiredAt", Instant.now())
                          .set("expiresAt", Instant.now().plus(ttl)),
                    LockDocument.class
                );
                return result.getModifiedCount() > 0;
            }
            return false;
        }
    }

    public void release(String lockId, String owner) {
        mongoTemplate.remove(
            Query.query(Criteria.where("_id").is(lockId).and("owner").is(owner)),
            LockDocument.class
        );
    }

    // Usage pattern
    public <T> T withLock(String lockId, Duration ttl, Duration waitTimeout, Supplier<T> action) {
        String owner = hostname() + "-" + Thread.currentThread().threadId();
        Instant deadline = Instant.now().plus(waitTimeout);

        while (Instant.now().isBefore(deadline)) {
            if (tryAcquire(lockId, owner, ttl)) {
                try {
                    return action.get();
                } finally {
                    release(lockId, owner);
                }
            }
            try { Thread.sleep(100); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new LockAcquisitionException("Interrupted waiting for lock");
            }
        }
        throw new LockAcquisitionException("Timeout waiting for lock: " + lockId);
    }
}

@Document("distributed_locks")
public record LockDocument(
    @Id String id,
    String owner,
    Instant acquiredAt,
    @Indexed(expireAfter = "0s") Instant expiresAt  // TTL auto-cleanup
) {}
```

---

## ⚡ Pattern 7: Performance Optimization

### Read Optimization

```java
// ── Projection — only load needed fields ──
var query = new Query()
    .addCriteria(Criteria.where("customerId").is(customerId))
    .fields()
        .include("id", "amount", "status", "createdAt")
        .exclude("statusHistory", "gatewayResponse");  // Skip large embedded data

// ── Covered query — answered entirely from index ──
// Index: { customerId: 1, status: 1, amount.value: 1 }
// Query only requests fields IN the index → no document fetch
var query = new Query()
    .addCriteria(Criteria.where("customerId").is(id).and("status").is("CAPTURED"))
    .fields().include("amount.value").exclude("_id");

// ── Batch reads with cursor ──
try (var cursor = mongoTemplate.stream(
        Query.query(Criteria.where("status").is("PENDING")),
        Payment.class)) {
    cursor.forEachRemaining(payment -> process(payment));
}

// ── Read preference for replicas ──
var query = new Query()
    .addCriteria(Criteria.where("status").is("CAPTURED"))
    .withReadPreference(ReadPreference.secondaryPreferred());
// Offloads reads from primary — eventual consistency OK for analytics
```

### Write Optimization

```java
// ── Bulk writes ──
var ops = mongoTemplate.bulkOps(BulkOperations.BulkMode.UNORDERED, Payment.class);
payments.forEach(p -> ops.insert(p));
var result = ops.execute();
// UNORDERED = parallel execution, faster but continues on error
// ORDERED   = sequential, stops on first error

// ── Partial updates (not full document replace) ──
// ❌ Slow: reads full doc, modifies, writes full doc back
var payment = mongoTemplate.findById(id, Payment.class);
var updated = payment.withStatus(CAPTURED);
mongoTemplate.save(updated);

// ✅ Fast: atomic field update, no full doc read
mongoTemplate.updateFirst(
    Query.query(Criteria.where("_id").is(id)),
    Update.update("status", "CAPTURED").set("updatedAt", Instant.now()),
    Payment.class
);

// ── Write concern tuning ──
// For critical data (payments):
mongoTemplate.setWriteConcern(WriteConcern.MAJORITY);
// For non-critical data (metrics, logs):
mongoTemplate.setWriteConcern(WriteConcern.W1);
```

### Key Performance Metrics to Monitor

```yaml
# MongoDB connection pool
mongodb.driver.pool.size           # Current pool size
mongodb.driver.pool.checkedout     # Active connections
mongodb.driver.pool.waitqueuesize  # Threads waiting for connection

# Operation timing
mongodb.driver.commands.timer      # Per-operation latency (find, insert, update)

# Alert thresholds:
#   pool.waitqueuesize > 0 for > 30s  → Pool too small
#   commands.timer p99 > 500ms        → Slow queries
#   pool.checkedout = pool.size       → Pool exhaustion
```

---

## 🔄 Pattern 8: Change Streams

> **Use case:** React to MongoDB changes in real-time — alternative to outbox polling.

```java
@Component
public class PaymentChangeStreamListener {

    private final MongoTemplate mongoTemplate;
    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    @PostConstruct
    public void startWatching() {
        var pipeline = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("operationType").in("insert", "update")
                .and("fullDocument.status").in("CAPTURED", "FAILED", "REFUNDED"))
        );

        Executors.newSingleThreadExecutor().submit(() -> {
            mongoTemplate.getCollection("payments")
                .watch(pipeline.getPipeline())
                .fullDocument(FullDocument.UPDATE_LOOKUP)
                .forEach(change -> {
                    var doc = change.getFullDocument();
                    if (doc != null) {
                        var event = toPaymentEvent(doc);
                        kafkaTemplate.send("payments.changes", doc.getString("_id"), event);
                    }
                });
        });
    }
}
```

---

## 🚫 MongoDB Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Unbounded arrays** | Document exceeds 16MB, slow updates | Bucket pattern or separate collection |
| **No indexes for queries** | Collection scans on large data | Analyze queries, add compound indexes |
| **Too many indexes** | Slow writes, high memory | Remove unused indexes, use compound |
| **Full doc read-modify-write** | Race conditions, unnecessary I/O | Use atomic `$set`, `$inc`, `$push` |
| **Transactions for single doc** | Unnecessary overhead | Single doc ops are already atomic |
| **Auto-index in production** | Blocks writes during index build | Create indexes during maintenance |
| **Ignoring write concern** | Data loss on replica failure | `w: majority` for critical data |
| **No TTL on temp data** | Collections grow forever | TTL indexes for events, sessions, caches |
| **Joining everything with $lookup** | MongoDB isn't a relational DB | Embed data, denormalize reads |
| **Large embedded docs rarely accessed** | Wastes memory and bandwidth | Use projection or separate collection |

---

## 💡 Golden Rules of MongoDB with Spring Data

```
1.  DESIGN FOR ACCESS PATTERNS — model documents around how you query, not how you think relationally.
2.  EMBED what you read together — reference what you update independently.
3.  INDEX follows QUERY — ESR rule: Equality → Sort → Range.
4.  ATOMIC UPDATES over read-modify-write — use $set, $inc, $push for field-level changes.
5.  BOUNDED ARRAYS only — cap embedded lists, overflow to separate collections.
6.  PROJECTION always — never load fields you don't need, especially large embedded objects.
7.  TRANSACTIONS sparingly — single doc is already atomic, use tx only for multi-doc consistency.
8.  MONITOR CONNECTION POOL — pool exhaustion is the #1 production MongoDB issue.
9.  TTL INDEXES for ephemeral data — processed events, sessions, locks, caches.
10. EXPLAIN your queries — if it says COLLSCAN on a large collection, you have a problem.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Spring Data MongoDB / MongoDB 7.x*
