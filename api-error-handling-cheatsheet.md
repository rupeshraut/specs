# 🛡️ API Error Handling & Resilience Cheat Sheet

> **Purpose:** Production patterns for handling errors, retries, rate limiting, and graceful degradation at the API layer. Reference before designing any error strategy or client-facing resilience mechanism.
> **Stack context:** Java 21+ / Spring Boot 3.x / Resilience4j / REST / Kafka

---

## 📋 The Error Handling Decision Framework

For every error scenario, answer:

| Question | Determines |
|----------|-----------|
| Is this the **client's fault** or **our fault**? | 4xx vs 5xx |
| Can the client **fix it and retry**? | Error message clarity |
| Is the failure **transient**? | Retry-After header |
| Should the client **wait and retry**? | 429/503 + Retry-After |
| Is there a **degraded response** we can give? | Fallback vs hard fail |
| Does the client need to **correlate** this with logs? | traceId in response |

---

## 🏗️ Pattern 1: Error Classification Taxonomy

### Three-Tier Error Model

```
TIER 1: CLIENT ERRORS (4xx) — "You sent something wrong"
  The client can fix the request and retry successfully.
  ┌────────────────────────────────────────────────────────────┐
  │ 400 — Malformed request (bad JSON, wrong types)            │
  │ 401 — Not authenticated (missing/expired token)            │
  │ 403 — Not authorized (valid token, wrong permissions)      │
  │ 404 — Resource doesn't exist                               │
  │ 409 — Conflict (duplicate, version mismatch)               │
  │ 422 — Business rule violation (valid format, bad logic)    │
  │ 429 — Rate limited (too many requests)                     │
  └────────────────────────────────────────────────────────────┘

TIER 2: TRANSIENT SERVER ERRORS (5xx) — "We failed, but it's temporary"
  The client should wait and retry.
  ┌────────────────────────────────────────────────────────────┐
  │ 502 — Upstream dependency failed                           │
  │ 503 — Service temporarily unavailable (overloaded/maint)   │
  │ 504 — Upstream dependency timed out                        │
  └────────────────────────────────────────────────────────────┘

TIER 3: PERMANENT SERVER ERRORS (5xx) — "We failed, retrying won't help"
  The client should NOT retry. Report to support.
  ┌────────────────────────────────────────────────────────────┐
  │ 500 — Unhandled exception (bug in our code)                │
  │ 501 — Not implemented                                      │
  └────────────────────────────────────────────────────────────┘
```

### Error Code Registry

```java
public enum ApiErrorCode {
    // ── Validation (400) ──
    INVALID_REQUEST("Invalid request format", HttpStatus.BAD_REQUEST, false),
    FIELD_REQUIRED("Required field is missing", HttpStatus.BAD_REQUEST, false),
    FIELD_INVALID("Field value is invalid", HttpStatus.BAD_REQUEST, false),

    // ── Authentication (401) ──
    AUTH_TOKEN_MISSING("Authentication token is required", HttpStatus.UNAUTHORIZED, false),
    AUTH_TOKEN_EXPIRED("Authentication token has expired", HttpStatus.UNAUTHORIZED, false),
    AUTH_TOKEN_INVALID("Authentication token is invalid", HttpStatus.UNAUTHORIZED, false),

    // ── Authorization (403) ──
    INSUFFICIENT_PERMISSIONS("You don't have permission", HttpStatus.FORBIDDEN, false),

    // ── Not Found (404) ──
    RESOURCE_NOT_FOUND("The requested resource was not found", HttpStatus.NOT_FOUND, false),

    // ── Conflict (409) ──
    DUPLICATE_RESOURCE("Resource with this identifier already exists", HttpStatus.CONFLICT, false),
    VERSION_CONFLICT("Resource was modified by another request", HttpStatus.CONFLICT, true),

    // ── Business Rules (422) ──
    PAYMENT_DECLINED("Payment was declined by the issuer", HttpStatus.UNPROCESSABLE_ENTITY, false),
    INSUFFICIENT_FUNDS("Insufficient funds for this transaction", HttpStatus.UNPROCESSABLE_ENTITY, false),
    AMOUNT_EXCEEDS_LIMIT("Amount exceeds the allowed limit", HttpStatus.UNPROCESSABLE_ENTITY, false),
    INVALID_STATE_TRANSITION("Action not allowed in current state", HttpStatus.UNPROCESSABLE_ENTITY, false),

    // ── Rate Limiting (429) ──
    RATE_LIMITED("Too many requests", HttpStatus.TOO_MANY_REQUESTS, true),

    // ── Server Errors (5xx) ──
    INTERNAL_ERROR("An unexpected error occurred", HttpStatus.INTERNAL_SERVER_ERROR, false),
    DOWNSTREAM_FAILURE("A required service is temporarily unavailable", HttpStatus.BAD_GATEWAY, true),
    SERVICE_UNAVAILABLE("Service is temporarily unavailable", HttpStatus.SERVICE_UNAVAILABLE, true),
    GATEWAY_TIMEOUT("A required service did not respond in time", HttpStatus.GATEWAY_TIMEOUT, true);

    private final String defaultMessage;
    private final HttpStatus httpStatus;
    private final boolean retryable;

    public boolean isRetryable() { return retryable; }
}
```

---

## 🔄 Pattern 2: Retry Guidance for API Clients

### Retry-After Headers

```java
@RestControllerAdvice
public class RetryGuidanceHandler {

    // ── Rate limited: tell client exactly when to retry ──
    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<ApiError> handleRateLimit(RateLimitExceededException ex,
            HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", String.valueOf(ex.retryAfterSeconds()))
            .header("X-RateLimit-Limit", String.valueOf(ex.limit()))
            .header("X-RateLimit-Remaining", "0")
            .header("X-RateLimit-Reset", String.valueOf(ex.resetEpochSeconds()))
            .body(createError("RATE_LIMITED",
                "Rate limit exceeded. Retry after %d seconds".formatted(ex.retryAfterSeconds()),
                request));
    }

    // ── Service unavailable: suggest retry with backoff ──
    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<ApiError> handleUnavailable(ServiceUnavailableException ex,
            HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "30")
            .body(createError("SERVICE_UNAVAILABLE",
                "Service temporarily unavailable. Please retry.", request));
    }

    // ── Conflict with optimistic locking: re-fetch and retry ──
    @ExceptionHandler(OptimisticLockingException.class)
    public ResponseEntity<ApiError> handleConflict(OptimisticLockingException ex,
            HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ApiError(
                "VERSION_CONFLICT",
                "Resource was modified. Re-fetch and retry with the new version.",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(),
                List.of(new ApiError.FieldError(
                    "version", "STALE",
                    "Expected version %d but current is %d".formatted(
                        ex.expectedVersion(), ex.actualVersion())))
            ));
    }
}
```

### Client Retry Decision Matrix

```
┌─────────┬──────────────┬────────────────────────────────────┐
│ Status  │ Should Retry │ How                                │
├─────────┼──────────────┼────────────────────────────────────┤
│ 400     │ ❌ No        │ Fix the request                    │
│ 401     │ ⚠️ Once     │ Refresh token, then retry ONCE     │
│ 403     │ ❌ No        │ Check permissions                  │
│ 404     │ ❌ No        │ Resource doesn't exist             │
│ 409     │ ✅ Yes       │ Re-fetch, retry with new version   │
│ 422     │ ❌ No        │ Business rule — change request     │
│ 429     │ ✅ Yes       │ Wait Retry-After seconds           │
│ 500     │ ⚠️ Once     │ Retry once; if persists, stop      │
│ 502     │ ✅ Yes       │ Exponential backoff (1s, 2s, 4s)   │
│ 503     │ ✅ Yes       │ Wait Retry-After or backoff        │
│ 504     │ ✅ Yes       │ Exponential backoff                │
└─────────┴──────────────┴────────────────────────────────────┘

RETRY ALGORITHM (for retryable errors):
  max_retries = 3
  for attempt in 1..max_retries:
    if response has Retry-After header:
      wait(Retry-After)
    else:
      delay = min(base_delay × 2^attempt + random_jitter, max_delay)
      wait(delay)
    retry with SAME Idempotency-Key
```

---

## 🚦 Pattern 3: Rate Limiting

### Server-Side Rate Limiter

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {

        String clientId = extractClientId(request);
        var limiter = limiters.computeIfAbsent(clientId,
            id -> RateLimiter.of(id, RateLimiterConfig.custom()
                .limitForPeriod(100)
                .limitRefreshPeriod(Duration.ofMinutes(1))
                .timeoutDuration(Duration.ZERO)
                .build()));

        if (limiter.acquirePermission()) {
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining",
                String.valueOf(limiter.getMetrics().getAvailablePermissions()));
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.setContentType("application/json");
            response.setHeader("Retry-After", "60");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.getWriter().write("""
                {"code":"RATE_LIMITED","message":"Rate limit exceeded. Retry after 60 seconds."}
                """);
        }
    }

    private String extractClientId(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) return "api:" + apiKey;
        var principal = request.getUserPrincipal();
        if (principal != null) return "user:" + principal.getName();
        return "ip:" + request.getRemoteAddr();
    }
}
```

### Tiered Rate Limit Configuration

```yaml
rate-limits:
  tiers:
    free:
      requests-per-minute: 60
      requests-per-day: 1000
    standard:
      requests-per-minute: 300
      requests-per-day: 50000
    enterprise:
      requests-per-minute: 3000
      requests-per-day: 1000000
  # Per-endpoint overrides
  endpoints:
    "POST /api/v1/payments":
      requests-per-minute: 30      # Writes are expensive
    "GET /api/v1/payments":
      requests-per-minute: 600     # Reads are cheap
```

---

## 🔀 Pattern 4: Graceful Degradation

### Degradation Strategies

```
STRATEGY 1: FALLBACK RESPONSE
  When:    Downstream enrichment service is down
  Action:  Return core data without enrichment
  Example: Return payment without fraud score detail

STRATEGY 2: CACHED RESPONSE
  When:    Primary data source is slow/down
  Action:  Return stale but valid cached data
  Example: Return cached exchange rate (5 min stale)

STRATEGY 3: FEATURE DISABLE
  When:    Non-critical feature dependency is down
  Action:  Disable the feature, continue with core
  Example: Skip loyalty points calculation

STRATEGY 4: QUEUE FOR LATER
  When:    Non-blocking operation fails
  Action:  Queue it for retry, return success to caller
  Example: Accept payment, queue notification for later

STRATEGY 5: PARTIAL RESPONSE
  When:    Some items in a batch operation fail
  Action:  Return successes + failures in one response
  Example: Batch payment — 8/10 succeeded, 2 failed
```

### Fallback Implementation

```java
@Service
public class EnrichedPaymentService {

    private final PaymentRepository repository;
    private final FraudService fraudService;
    private final CircuitBreaker fraudCircuitBreaker;

    public PaymentDetailResponse getPaymentDetail(String paymentId) {
        var payment = repository.findById(paymentId)
            .orElseThrow(() -> new ResourceNotFoundException("Payment", paymentId));

        // Enrichment with fallback
        FraudDetail fraudDetail = getFraudDetailWithFallback(paymentId);

        return new PaymentDetailResponse(
            payment.id(),
            payment.amount(),
            payment.status(),
            fraudDetail   // May be null or degraded
        );
    }

    private FraudDetail getFraudDetailWithFallback(String paymentId) {
        try {
            return CircuitBreaker.decorateSupplier(fraudCircuitBreaker,
                () -> fraudService.getDetail(paymentId)).get();
        } catch (CallNotPermittedException e) {
            log.warn("Fraud service circuit open — returning degraded response");
            return FraudDetail.unavailable("Fraud detail temporarily unavailable");
        } catch (Exception e) {
            log.warn("Fraud service call failed: {}", e.getMessage());
            return FraudDetail.unavailable("Fraud detail could not be retrieved");
        }
    }
}
```

### Partial Success (Batch) Response Pattern

```java
public record BatchResponse<T>(
    String batchId,
    int totalRequested,
    int succeeded,
    int failed,
    List<BatchItemResult<T>> results
) {
    public record BatchItemResult<T>(
        String requestId,
        String status,        // "SUCCEEDED" or "FAILED"
        T data,               // null if failed
        ApiError error        // null if succeeded
    ) {}
}

@PostMapping("/payments/batch")
public ResponseEntity<BatchResponse<PaymentResponse>> createBatch(
        @RequestBody @Valid BatchPaymentRequest request) {

    var results = request.payments().stream()
        .map(item -> {
            try {
                var result = paymentService.create(item);
                return new BatchResponse.BatchItemResult<>(
                    item.requestId(), "SUCCEEDED", mapper.toResponse(result), null);
            } catch (BusinessRuleException e) {
                return new BatchResponse.BatchItemResult<PaymentResponse>(
                    item.requestId(), "FAILED", null,
                    new ApiError(e.errorCode(), e.getMessage(), null, null, null, null));
            }
        })
        .toList();

    long succeeded = results.stream().filter(r -> "SUCCEEDED".equals(r.status())).count();
    long failed = results.size() - succeeded;

    var response = new BatchResponse<>(
        UUID.randomUUID().toString(),
        results.size(), (int) succeeded, (int) failed, results);

    // 200 all good, 207 mixed, 422 all failed
    HttpStatus status = failed == 0 ? HttpStatus.OK
        : succeeded == 0 ? HttpStatus.UNPROCESSABLE_ENTITY
        : HttpStatus.MULTI_STATUS;

    return ResponseEntity.status(status).body(response);
}
```

---

## 🔌 Pattern 5: Circuit Breaker at API Boundary

### Mapping Resilience4j to HTTP Responses

```java
@RestControllerAdvice
public class ResilienceExceptionHandler {

    // Circuit breaker open → 503
    @ExceptionHandler(CallNotPermittedException.class)
    public ResponseEntity<ApiError> handleCircuitOpen(
            CallNotPermittedException ex, HttpServletRequest request) {
        log.warn("Circuit breaker OPEN: {}", ex.getCausingCircuitBreakerName());
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "30")
            .header("X-Circuit-Breaker", ex.getCausingCircuitBreakerName())
            .body(new ApiError(
                "SERVICE_UNAVAILABLE",
                "Service temporarily unavailable. Please retry after 30 seconds.",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // Bulkhead full → 503
    @ExceptionHandler(BulkheadFullException.class)
    public ResponseEntity<ApiError> handleBulkheadFull(
            BulkheadFullException ex, HttpServletRequest request) {
        log.warn("Bulkhead full: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "5")
            .body(new ApiError(
                "SERVICE_OVERLOADED",
                "Service is at capacity. Please retry shortly.",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // Timeout → 504
    @ExceptionHandler(TimeoutException.class)
    public ResponseEntity<ApiError> handleTimeout(
            TimeoutException ex, HttpServletRequest request) {
        log.error("Timeout: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.GATEWAY_TIMEOUT)
            .header("Retry-After", "10")
            .body(new ApiError(
                "GATEWAY_TIMEOUT",
                "Request timed out. Please retry.",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // Retry exhausted → 502
    @ExceptionHandler(MaxRetriesExceededException.class)
    public ResponseEntity<ApiError> handleRetryExhausted(
            MaxRetriesExceededException ex, HttpServletRequest request) {
        log.error("Retries exhausted: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_GATEWAY)
            .header("Retry-After", "30")
            .body(new ApiError(
                "DOWNSTREAM_FAILURE",
                "A required service is temporarily unavailable.",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }
}
```

---

## 📊 Pattern 6: Error Observability

### Error Metrics

```java
@Component
public class ApiErrorMetrics {

    private final MeterRegistry registry;

    public void recordError(String path, String method, int statusCode, String errorCode) {
        registry.counter("api.errors",
            "path", normalizePath(path),
            "method", method,
            "status", String.valueOf(statusCode),
            "error_code", errorCode
        ).increment();
    }

    // Normalize to avoid high cardinality
    private String normalizePath(String path) {
        return path.replaceAll("/[a-f0-9-]{8,}", "/{id}")
                   .replaceAll("/\\d+", "/{id}");
    }
}
```

### Error Response Logging Strategy

```java
// Log level by error tier:
//   Business rule (422)     → INFO   (expected, not bugs)
//   Client error (4xx)      → DEBUG  (client's problem)
//   Transient server (5xx)  → WARN   (worth watching)
//   Unexpected (500)        → ERROR  (potential bug, needs investigation)

@RestControllerAdvice
public class ErrorLoggingAdvice {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleAndLog(Exception ex, HttpServletRequest request) {
        var level = classifyLogLevel(ex);
        switch (level) {
            case INFO -> log.info("Business rule: code={}, path={}",
                getCode(ex), request.getRequestURI());
            case WARN -> log.warn("Transient failure: path={}, error={}",
                request.getRequestURI(), ex.getMessage());
            case ERROR -> log.error("Unhandled exception: path={}, method={}",
                request.getRequestURI(), request.getMethod(), ex);
        }
        return buildErrorResponse(ex, request);
    }
}
```

### Alert Rules for API Errors

```yaml
# Client error spike (possible attack or breaking change)
- alert: Client4xxSpike
  expr: >
    sum(rate(api_errors_total{status=~"4.."}[5m])) /
    sum(rate(http_server_requests_seconds_count[5m]))
    > 0.20
  for: 5m
  severity: ticket
  annotations:
    summary: "Client error rate is {{ $value | humanizePercentage }}"

# Server error rate
- alert: ServerErrorRate
  expr: >
    sum(rate(api_errors_total{status=~"5.."}[5m])) /
    sum(rate(http_server_requests_seconds_count[5m]))
    > 0.05
  for: 2m
  severity: page

# Specific business error spike
- alert: PaymentDeclineSpike
  expr: rate(api_errors_total{error_code="PAYMENT_DECLINED"}[5m]) > 10
  for: 5m
  severity: ticket
  annotations:
    summary: "Payment decline rate spiked to {{ $value }}/s"

# Rate limit triggers increasing
- alert: RateLimitTriggersHigh
  expr: rate(api_errors_total{error_code="RATE_LIMITED"}[5m]) > 5
  for: 5m
  severity: ticket
```

---

## 🔐 Pattern 7: Security Error Handling

### Authentication/Authorization Error Responses

```java
@Component
public class SecurityExceptionHandler extends AuthenticationEntryPoint
        implements AccessDeniedHandler {

    // ── 401: Not authenticated ──
    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException ex) throws IOException {
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("application/json");
        response.getWriter().write(new ObjectMapper().writeValueAsString(new ApiError(
            "AUTH_TOKEN_INVALID",
            "Authentication required. Provide a valid Bearer token.",
            request.getHeader("X-Trace-Id"),
            Instant.now(),
            request.getRequestURI(),
            null
        )));
    }

    // ── 403: Not authorized ──
    @Override
    public void handle(HttpServletRequest request,
            HttpServletResponse response, AccessDeniedException ex) throws IOException {
        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType("application/json");
        response.getWriter().write(new ObjectMapper().writeValueAsString(new ApiError(
            "INSUFFICIENT_PERMISSIONS",
            "You do not have permission to perform this action.",
            request.getHeader("X-Trace-Id"),
            Instant.now(),
            request.getRequestURI(),
            null
        )));
    }
}
```

### Security Error Rules

```
⚠️ NEVER reveal in error responses:
  • Whether a resource EXISTS (use generic 404 for auth failures too)
  • Internal system names (database names, service names)
  • Stack traces or file paths
  • SQL/NoSQL queries
  • Which authentication step failed (just "invalid credentials")

✅ ALWAYS include:
  • traceId for support correlation
  • Generic but helpful message
  • Correct HTTP status code
  • Error code for programmatic handling
```

---

## 🚫 API Error Handling Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **200 OK with error body** | Client can't distinguish success from failure | Use proper HTTP status codes |
| **Leaking stack traces** | Reveals internal implementation to attackers | Generic messages + traceId |
| **Different error shapes** | Each endpoint returns different error format | Single ApiError format everywhere |
| **Swallowing exceptions** | Hides bugs, makes debugging impossible | Log all errors, return appropriate code |
| **No traceId in errors** | Support can't correlate with logs | Include traceId in every error response |
| **Retryable without guidance** | Client doesn't know when/how to retry | Retry-After header on 429/503 |
| **500 for business rules** | Triggers wrong alerts, wrong client behavior | 422 for business rules, 500 for bugs |
| **No rate limiting** | DoS vulnerability, resource exhaustion | Rate limit all public endpoints |
| **Hard fail on enrichment** | Non-critical failure kills the entire request | Fallback/degrade for optional data |
| **Logging PII in errors** | Security/compliance breach | Mask PII before logging |
| **No circuit breaker mapping** | Resilience4j exceptions leak to clients | Map to 503/504 with Retry-After |
| **Catch-all returning 400** | Server errors miscategorized as client errors | Separate 4xx and 5xx handling |

---

## 💡 Golden Rules of API Error Handling

```
1.  EVERY error response has the SAME shape — ApiError with code, message, traceId, timestamp.
2.  HTTP status codes are PRECISE — 400 ≠ 422 ≠ 409. Each means something different.
3.  Error codes are MACHINE-READABLE — clients switch on error codes, humans read messages.
4.  NEVER expose internals — no stack traces, no SQL, no service names in client responses.
5.  TRANSIENT errors get Retry-After — tell clients WHEN and HOW to retry.
6.  PERMANENT errors get clear guidance — tell clients WHAT to fix.
7.  traceId bridges the gap — between the error the client sees and the logs you debug.
8.  DEGRADE gracefully — a partial answer beats no answer.
9.  LOG at the right level — INFO for business rules, WARN for transient, ERROR for bugs.
10. RATE LIMIT everything — the internet is adversarial.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Resilience4j / REST*
