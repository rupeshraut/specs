# 🔐 Distributed Locking & Coordination Cheat Sheet

> **Purpose:** Production patterns for distributed locks, leader election, and coordination across service instances.
> **Stack context:** Java 21+ / Spring Boot 3.x / MongoDB / Redis

---

## 📋 When Do You Need Distributed Locking?

| Scenario | Pattern |
|----------|---------|
| Prevent duplicate payment processing | Idempotency key (preferred) or lock |
| Only one instance runs scheduled job | Leader election |
| Exclusive access to external resource | Distributed lock with TTL |
| Ordered processing of related events | Kafka partition key (no lock needed) |

---

## 🔒 Pattern 1: MongoDB Distributed Lock

```java
@Document("distributed_locks")
public record LockDocument(
    @Id String id,
    String owner,
    Instant acquiredAt,
    @Indexed(expireAfter = "0s") Instant expiresAt,
    long fenceToken
) {}

@Component
public class MongoDistributedLock {

    private final MongoTemplate mongoTemplate;
    private final String instanceId = hostname() + "-" + ProcessHandle.current().pid();

    public Optional<Lock> tryAcquire(String lockName, Duration ttl) {
        var now = Instant.now();
        try {
            var doc = mongoTemplate.insert(new LockDocument(
                lockName, instanceId, now, now.plus(ttl), System.nanoTime()));
            return Optional.of(new Lock(doc.id(), doc.owner(), doc.fenceToken()));
        } catch (DuplicateKeyException e) {
            var result = mongoTemplate.updateFirst(
                Query.query(Criteria.where("_id").is(lockName).and("expiresAt").lt(now)),
                Update.update("owner", instanceId)
                      .set("acquiredAt", now).set("expiresAt", now.plus(ttl)).inc("fenceToken", 1),
                LockDocument.class);
            if (result.getModifiedCount() > 0) {
                var doc = mongoTemplate.findById(lockName, LockDocument.class);
                return Optional.of(new Lock(doc.id(), doc.owner(), doc.fenceToken()));
            }
            return Optional.empty();
        }
    }

    public boolean release(String lockName) {
        return mongoTemplate.remove(
            Query.query(Criteria.where("_id").is(lockName).and("owner").is(instanceId)),
            LockDocument.class).getDeletedCount() > 0;
    }

    public <T> T withLock(String lockName, Duration ttl, Duration timeout, Supplier<T> action) {
        var deadline = Instant.now().plus(timeout);
        while (Instant.now().isBefore(deadline)) {
            var lock = tryAcquire(lockName, ttl);
            if (lock.isPresent()) {
                try { return action.get(); }
                finally { release(lockName); }
            }
            try { Thread.sleep(100); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new LockAcquisitionException("Interrupted");
            }
        }
        throw new LockAcquisitionException("Timeout acquiring lock: " + lockName);
    }

    public boolean extend(String lockName, Duration additionalTtl) {
        return mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is(lockName).and("owner").is(instanceId)),
            Update.update("expiresAt", Instant.now().plus(additionalTtl)),
            LockDocument.class).getModifiedCount() > 0;
    }

    public record Lock(String name, String owner, long fenceToken) {}
}
```

### Fencing Tokens

```
PROBLEM: Instance A acquires lock → pauses (GC) → lock expires → Instance B acquires →
         Instance A resumes → BOTH think they hold the lock

SOLUTION: Monotonic fenceToken included in all downstream writes
  Instance A (token=1) → write(token=1) ✅
  Instance B (token=2) → write(token=2) ✅
  Instance A (token=1) → write(token=1) ❌ REJECTED (stale)
```

---

## 👑 Pattern 2: Leader Election

```java
@Component
public class MongoLeaderElection {

    private final MongoTemplate mongoTemplate;
    private final String instanceId = hostname() + "-" + ProcessHandle.current().pid();
    private volatile boolean isLeader = false;
    private static final Duration TERM = Duration.ofSeconds(30);

    @Scheduled(fixedDelay = 10000)
    public void heartbeat() {
        isLeader = isLeader ? renewLeadership() : tryBecomeLeader();
    }

    private boolean tryBecomeLeader() {
        try {
            mongoTemplate.insert(new LeaderDocument(
                "singleton-leader", instanceId, Instant.now(), Instant.now().plus(TERM)));
            log.info("Became leader: {}", instanceId);
            return true;
        } catch (DuplicateKeyException e) {
            var result = mongoTemplate.updateFirst(
                Query.query(Criteria.where("_id").is("singleton-leader")
                        .and("expiresAt").lt(Instant.now())),
                Update.update("owner", instanceId)
                      .set("expiresAt", Instant.now().plus(TERM)),
                LeaderDocument.class);
            return result.getModifiedCount() > 0;
        }
    }

    private boolean renewLeadership() {
        return mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").is("singleton-leader")
                    .and("owner").is(instanceId)),
            Update.update("expiresAt", Instant.now().plus(TERM)),
            LeaderDocument.class).getModifiedCount() > 0;
    }

    public boolean isLeader() { return isLeader; }

    @PreDestroy
    void relinquish() {
        if (isLeader) {
            mongoTemplate.remove(Query.query(Criteria.where("_id").is("singleton-leader")
                    .and("owner").is(instanceId)), LeaderDocument.class);
        }
    }
}

// Usage: only leader runs the outbox poller
@Scheduled(fixedDelay = 100)
public void poll() {
    if (leaderElection.isLeader()) outboxPoller.pollAndPublish();
}
```

---

## 🚦 Pattern 3: Distributed Semaphore

```java
@Component
public class MongoDistributedSemaphore {

    private final MongoTemplate mongoTemplate;

    public boolean tryAcquire(String resource, int maxPermits, String holder, Duration ttl) {
        var now = Instant.now();
        // Clean expired permits
        mongoTemplate.remove(Query.query(Criteria.where("resource").is(resource)
                .and("expiresAt").lt(now)), SemaphorePermit.class);

        long active = mongoTemplate.count(
            Query.query(Criteria.where("resource").is(resource)), SemaphorePermit.class);

        if (active >= maxPermits) return false;

        try {
            mongoTemplate.insert(new SemaphorePermit(
                UUID.randomUUID().toString(), resource, holder, now, now.plus(ttl)));
            return true;
        } catch (Exception e) {
            return false; // Race condition — another instance grabbed the last permit
        }
    }

    public void release(String resource, String holder) {
        mongoTemplate.remove(Query.query(Criteria.where("resource").is(resource)
                .and("holder").is(holder)), SemaphorePermit.class);
    }
}
```

---

## 🚫 Distributed Locking Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Lock without TTL | Always set expiry — crashed holder = permanent deadlock |
| No fencing token | Stale lock holder corrupts data |
| Lock too long | Short TTL + extend pattern |
| Ignoring clock skew | Use relative TTLs, not absolute timestamps across machines |
| Lock for performance | Locks are for correctness — use caching for performance |
| Not handling lock loss | Check lock validity before critical writes |

---

## 💡 Golden Rules

```
1.  IDEMPOTENCY > LOCKING — if you can make the operation idempotent, you don't need a lock.
2.  TTL on every lock — no TTL = potential permanent deadlock.
3.  FENCING TOKENS prevent stale holders — include in all downstream writes.
4.  LEADER ELECTION for singleton tasks — don't run scheduled jobs on all instances.
5.  EXTEND, don't set long TTLs — short TTL + heartbeat extension is safer.
6.  ASSUME the lock can be lost — always validate before critical writes.
7.  MongoDB locks are "good enough" — don't add Redis just for locking.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / MongoDB*
