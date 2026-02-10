# 🚨 Incident Response & Runbook Template Cheat Sheet

> **Purpose:** Standardized incident response playbook, investigation framework, and runbook templates for distributed Java services. Reference during any production incident or when writing runbooks.
> **Stack context:** Java 21+ / Spring Boot 3.x / MongoDB / Kafka / Kubernetes

---

## 📋 Incident Severity Levels

| Severity | Definition | Response Time | Example |
|----------|-----------|--------------|---------|
| **SEV-1** | Complete outage, data loss, financial impact | Immediate (page) | Payment processing down |
| **SEV-2** | Major degradation, significant user impact | 15 min (page) | 50% error rate, high latency |
| **SEV-3** | Minor degradation, limited impact | 1 hour (ticket) | Non-critical feature broken |
| **SEV-4** | Cosmetic, monitoring anomaly | Next business day | Dashboard gap, log noise |

---

## 🔄 Incident Response Flow

```
 DETECT (Alert / Customer report / Monitoring)
    │
    ▼
 TRIAGE (2 min)
    • Confirm the issue is real (not false alarm)
    • Assess severity (SEV 1-4)
    • Assign incident commander
    │
    ▼
 COMMUNICATE (5 min)
    • Open incident channel (#inc-PAY-2026-02-09)
    • Post initial status to stakeholders
    • "We are investigating elevated error rates on payment processing"
    │
    ▼
 INVESTIGATE (parallel tracks)
    ├── Check dashboards (golden signals: rate, errors, latency, saturation)
    ├── Check recent deployments (last 24h)
    ├── Check dependent services (MongoDB, Kafka, gateway)
    ├── Check infrastructure (K8s pods, CPU, memory, disk)
    └── Check logs (filtered by traceId, error level, timeframe)
    │
    ▼
 MITIGATE (stop the bleeding)
    • Rollback deployment if change-related
    • Toggle feature flag off
    • Scale up if capacity-related
    • Fail over to backup if dependency-related
    • Restart pods if memory leak / stuck threads
    │
    ▼
 RESOLVE (confirm recovery)
    • Verify metrics return to normal
    • Verify end-to-end health check passes
    • Monitor for 15-30 minutes post-fix
    │
    ▼
 POST-MORTEM (within 48 hours)
    • Document timeline, root cause, impact
    • Identify action items
    • Share learnings
```

---

## 🔍 Investigation Playbook

### Step 1: Golden Signals Check (2 minutes)

```
Dashboard: Service Health

TRAFFIC:    Is request rate normal?
  Low → upstream issue, DNS, load balancer
  High → traffic spike, DDoS, retry storm

ERRORS:     What's the error rate?
  Sudden spike → deployment, config change, dependency failure
  Gradual rise → resource exhaustion, data growth

LATENCY:    What are p50/p95/p99?
  All elevated → downstream dependency slow
  p99 only → specific code path or data issue

SATURATION: Resource usage?
  CPU high → computation issue, infinite loop, GC thrashing
  Memory high → leak, cache growth, large payloads
  Connections full → pool exhaustion, slow queries
```

### Step 2: Recent Changes (3 minutes)

```bash
# What deployed recently?
kubectl rollout history deployment/payment-service

# What changed in config?
kubectl get configmap payment-config -o yaml
kubectl get events --sort-by=.lastTimestamp | head -20

# Any infrastructure changes?
# Check: Kafka cluster, MongoDB replica set, network policies
```

### Step 3: Service-Specific Diagnosis

```bash
# ── Pod status ──
kubectl get pods -l app=payment-service
kubectl describe pod <pod-name>    # Check events, OOMKilled, CrashLoopBackOff

# ── Logs (last 5 minutes, errors only) ──
kubectl logs -l app=payment-service --since=5m | grep -i error | head -50

# ── Logs with traceId ──
kubectl logs -l app=payment-service --since=5m | grep "traceId=abc-123"

# ── JVM diagnostics (exec into pod) ──
kubectl exec -it <pod> -- jcmd 1 GC.heap_dump /tmp/heap.hprof
kubectl exec -it <pod> -- jcmd 1 Thread.print
kubectl exec -it <pod> -- jcmd 1 VM.flags

# ── Actuator endpoints ──
kubectl port-forward <pod> 8081:8081
curl localhost:8081/actuator/health
curl localhost:8081/actuator/metrics/jvm.memory.used
curl localhost:8081/actuator/metrics/http.server.requests
```

---

## 📖 Runbook Templates

### Runbook: High Error Rate

```markdown
## Alert: HighErrorRate (> 5% for 2+ minutes)

### Quick Check
1. Open Grafana: [Payment Service Dashboard]
2. Check: Which endpoints have elevated errors?
3. Check: Is it all instances or one pod?

### Common Causes & Actions

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| 5xx on all endpoints | Bad deployment | `kubectl rollout undo deployment/payment-service` |
| 5xx on one endpoint | Code bug | Check logs for stack trace, hotfix or feature-flag off |
| 502/504 errors | Downstream failure | Check MongoDB/Kafka/gateway status |
| 503 errors | Circuit breaker open | Check downstream health, wait for half-open |
| OOMKilled pods | Memory leak or undersized | Increase memory limits, capture heap dump |

### If MongoDB is the issue:
1. Check: `curl localhost:8081/actuator/health` → mongo status
2. Check: MongoDB connection pool: `curl localhost:8081/actuator/metrics/mongodb.driver.pool.checkedout`
3. Check: MongoDB Atlas/cluster status
4. Action: If pool exhausted, restart pods (releases connections)

### If Kafka is the issue:
1. Check: Consumer lag: `kafka-consumer-groups --describe --group payment-service`
2. Check: Broker health
3. Action: If consumer stuck, restart consumer pod

### Escalation
If not resolved in 15 minutes: page on-call lead
```

### Runbook: High Latency

```markdown
## Alert: HighP99Latency (> 2s for 5+ minutes)

### Quick Check
1. Which endpoint is slow? (check per-endpoint latency breakdown)
2. Is it one pod or all pods? (check per-instance)
3. What's the GC pause time? (check jvm_gc_pause_seconds_max)

### Common Causes & Actions

| Symptom | Likely Cause | Action |
|---------|-------------|--------|
| All endpoints slow | DB or downstream slow | Check MongoDB query times |
| One endpoint slow | Slow query or N+1 | Check logs for slow query warnings |
| GC pauses > 200ms | Heap pressure | Check heap usage, increase memory |
| Spiky latency | Lock contention | Thread dump: `jcmd 1 Thread.print` |
| Gradual increase | Data growth, missing index | Check MongoDB slow query log |

### MongoDB slow query check:
db.system.profile.find({millis: {$gt: 100}}).sort({ts: -1}).limit(10)
```

### Runbook: Kafka Consumer Lag

```markdown
## Alert: KafkaConsumerLag (> 10,000 for 10+ minutes)

### Quick Check
1. Which topic/partition has lag?
2. Is the consumer running? (`curl localhost:8081/actuator/health` → kafka status)
3. Is it processing but slow, or completely stuck?

### Actions
1. **Consumer not running:**
   - Check pod status: `kubectl get pods`
   - Check logs for deserialization errors (poison pill)
   - Restart: `kubectl rollout restart deployment/payment-service`

2. **Consumer slow:**
   - Check processing time per message
   - Check downstream dependencies (DB, gateway)
   - Scale up consumer instances (max = partition count)

3. **Stuck on one message (poison pill):**
   - Check DLT topic for recent messages
   - If no DLT, check error logs for repeated exceptions
   - Skip message: adjust consumer offset manually (last resort)
   
4. **Lag from traffic spike:**
   - Scale consumers (if < partition count)
   - Increase batch size temporarily
   - Monitor: lag should decrease once throughput > ingest rate
```

### Runbook: Circuit Breaker Open

```markdown
## Alert: CircuitBreakerOpen

### Quick Check
1. Which circuit breaker? (check X-Circuit-Breaker header or metrics)
2. Is the downstream actually down?
3. When did it open? (check state transition logs)

### Actions
1. Check downstream service health directly
2. If downstream is recovering → wait for half-open → automatic recovery
3. If downstream is down:
   - Is there a fallback? → Verify fallback is working
   - No fallback → Communicate expected recovery time
4. If false positive (downstream is fine):
   - Check timeout configuration (might be too aggressive)
   - Check network connectivity between pods
   - May need to adjust failure rate threshold
```

---

## 📝 Post-Mortem Template

```markdown
# Post-Mortem: [Brief Title]

**Date:** YYYY-MM-DD
**Severity:** SEV-X
**Duration:** HH:MM (from detection to resolution)
**Author:** [Name]
**Reviewers:** [Names]

## Summary
One paragraph: what happened, who was impacted, how it was resolved.

## Impact
- **Users affected:** X customers
- **Revenue impact:** $X / X failed payments
- **Duration of impact:** X minutes
- **SLO impact:** Availability dropped to X% (budget: X% consumed)

## Timeline (all times in EST)
| Time | Event |
|------|-------|
| 14:00 | Deployment of v2.3.1 to production |
| 14:05 | Error rate alert fired (5% → 15%) |
| 14:07 | Incident commander assigned, channel opened |
| 14:10 | Identified: new code path throws NPE for legacy payment types |
| 14:12 | Mitigation: feature flag disabled for new payment flow |
| 14:15 | Error rate returned to normal |
| 14:45 | Root cause confirmed, hotfix PR opened |
| 15:30 | Hotfix deployed with fix + test coverage |
| 15:45 | Feature flag re-enabled, monitoring confirmed stable |

## Root Cause
The new idempotency check assumed all payments have a `gatewayResponse` field,
but legacy ACH payments have this as null. The NPE was thrown on ~30% of payments
(the ACH proportion).

## Detection
Automated alert on error rate > 5% (fired at 14:05, 5 minutes after deployment).

## Resolution
1. Immediate: disabled feature flag (2 min to resolve)
2. Permanent: added null check + test for all payment types

## What Went Well
- Alert fired quickly (5 min)
- Feature flag allowed instant mitigation
- Team responded within 2 minutes

## What Went Wrong
- No test coverage for ACH payment path with new code
- Code review didn't catch the null assumption
- No canary deployment (went to 100% immediately)

## Action Items
| Action | Owner | Priority | Deadline |
|--------|-------|----------|----------|
| Add integration test for all payment types with idempotency | [Name] | P1 | [Date] |
| Implement canary deployment (10% → 50% → 100%) | [Name] | P2 | [Date] |
| Add null-safety ArchUnit rule for gateway response | [Name] | P2 | [Date] |
| Review all new code paths for legacy payment compatibility | [Name] | P1 | [Date] |

## Lessons Learned
1. Feature flags are invaluable — this would have been a 30+ minute outage without one
2. Legacy data shapes must be tested — not all payments look the same
3. Canary deployments would have caught this at 10% traffic
```

---

## 💡 Golden Rules of Incident Response

```
1.  STOP THE BLEEDING first, investigate second — rollback > root cause analysis.
2.  COMMUNICATE early and often — silence is worse than "we're investigating."
3.  FEATURE FLAGS are your emergency brake — toggle off in seconds.
4.  RUNBOOKS for every alert — if there's no action, it's not an alert.
5.  POST-MORTEM is blameless — focus on systems, not people.
6.  ACTION ITEMS have owners and deadlines — or they don't happen.
7.  Every incident improves the system — add the test, the alert, the guard.
8.  PRACTICE incidents — game days, chaos engineering, tabletop exercises.
9.  Document as you go — the timeline is hardest to reconstruct later.
10. The best incident is the one that never reaches customers — invest in prevention.
```

---

*Last updated: February 2026*
