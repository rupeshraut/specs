# 🌐 REST API Design Cheat Sheet

> **Purpose:** Production-grade REST API design patterns for enterprise services. Reference before designing any endpoint, error response, or versioning strategy.
> **Stack context:** Java 21+ / Spring Boot 3.x / Spring Web / Jackson / OpenAPI

---

## 📋 API Design Decision Framework

Before creating any endpoint, answer:

| Question | Determines |
|----------|-----------|
| Is this a **resource** or an **action**? | CRUD endpoint vs custom action |
| Who are the **consumers**? (internal, public, mobile) | Auth, rate limiting, payload size |
| What's the **consistency** requirement? | Sync response vs async (202) |
| Can the operation be **retried** safely? | Idempotency design |
| What **data** does the consumer actually need? | Sparse fieldsets, pagination |
| How will this API **evolve**? | Versioning strategy |

---

## 🏗️ Pattern 1: Resource Naming & URL Design

### Naming Rules

```
✅ RULES:
  • Use NOUNS for resources, not verbs
  • Use PLURAL nouns
  • Use lowercase with hyphens
  • Nest resources to show relationships (max 2 levels)
  • Use query parameters for filtering, sorting, pagination

✅ GOOD:
  GET    /api/v1/payments
  GET    /api/v1/payments/{paymentId}
  POST   /api/v1/payments
  GET    /api/v1/payments/{paymentId}/refunds
  POST   /api/v1/payments/{paymentId}/refunds
  GET    /api/v1/customers/{customerId}/payments

❌ BAD:
  GET    /api/v1/getPayments              (verb in URL)
  POST   /api/v1/createPayment            (verb in URL)
  GET    /api/v1/payment                  (singular)
  GET    /api/v1/Payment                  (uppercase)
  GET    /api/v1/payment_list             (underscore)
  GET    /api/v1/customers/123/payments/456/refunds/789/items  (too deep)
```

### Actions That Don't Fit CRUD

```
For operations that aren't pure CRUD, use sub-resource actions:

POST /api/v1/payments/{id}/capture          ← Capture a payment
POST /api/v1/payments/{id}/cancel           ← Cancel a payment
POST /api/v1/payments/{id}/retry            ← Retry a failed payment
POST /api/v1/transfers                      ← Create a transfer (not a payment action)
POST /api/v1/reports/daily-summary          ← Trigger report generation

For bulk operations:
POST /api/v1/payments/batch                 ← Batch create
POST /api/v1/payments/batch-status          ← Batch status check
```

---

## 📬 Pattern 2: HTTP Methods & Status Codes

### Method Semantics

| Method | Semantics | Idempotent | Safe | Request Body | Response Body |
|--------|-----------|------------|------|-------------|--------------|
| **GET** | Read resource | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **POST** | Create / action | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **PUT** | Full replace | ✅ Yes | ❌ No | ✅ Yes | ✅ Yes |
| **PATCH** | Partial update | ❌ No* | ❌ No | ✅ Yes | ✅ Yes |
| **DELETE** | Remove resource | ✅ Yes | ❌ No | ❌ No | ⚠️ Optional |

*PATCH can be made idempotent with conditional updates (If-Match)

### Status Code Decision Tree

```
Was the request understood and valid?
├── NO → Is the problem with the request format?
│   ├── YES → 400 Bad Request (malformed JSON, missing field)
│   └── NO → Is it an auth problem?
│       ├── Not authenticated → 401 Unauthorized
│       ├── Not authorized   → 403 Forbidden
│       └── Resource issue?
│           ├── Not found     → 404 Not Found
│           ├── Conflict      → 409 Conflict (duplicate, version mismatch)
│           ├── Gone          → 410 Gone (was here, permanently removed)
│           └── Too large     → 413 Payload Too Large
│
└── YES → Did processing succeed?
    ├── YES → What happened?
    │   ├── Resource created        → 201 Created (+ Location header)
    │   ├── Accepted for async      → 202 Accepted (+ polling URL)
    │   ├── Successful, no body     → 204 No Content (DELETE, PUT)
    │   └── Successful, with body   → 200 OK
    │
    └── NO → Why did it fail?
        ├── Business rule violation  → 422 Unprocessable Entity
        ├── Rate limited            → 429 Too Many Requests (+ Retry-After)
        ├── Server error            → 500 Internal Server Error
        ├── Downstream failure      → 502 Bad Gateway
        ├── Service overloaded      → 503 Service Unavailable (+ Retry-After)
        └── Downstream timeout      → 504 Gateway Timeout
```

### Status Code Quick Reference

| Code | When to Use | Example |
|------|------------|---------|
| **200** | Successful read/update | GET payment, PATCH payment |
| **201** | Resource created | POST payment → return created payment + Location header |
| **202** | Async processing started | POST batch-payment → return job ID |
| **204** | Success, no response body | DELETE payment, PUT with no return |
| **400** | Malformed request | Invalid JSON, missing required field |
| **401** | No/invalid credentials | Missing/expired token |
| **403** | Authenticated but not allowed | User can't access this resource |
| **404** | Resource doesn't exist | GET /payments/non-existent-id |
| **409** | Conflict with current state | Duplicate idempotency key, version conflict |
| **422** | Valid request, business rule fail | Insufficient funds, amount exceeds limit |
| **429** | Rate limited | Too many requests, include Retry-After header |
| **500** | Unexpected server error | Unhandled exception |
| **503** | Temporarily unavailable | Circuit breaker open, maintenance |

---

## 📦 Pattern 3: Request & Response Design

### Request Records

```java
// ── Create request ──
public record CreatePaymentRequest(
    @NotBlank String customerId,
    @NotNull @Positive BigDecimal amount,
    @NotNull @Size(min = 3, max = 3) String currency,
    @NotBlank String paymentType,
    @Valid PaymentMethodRequest paymentMethod,
    @Size(max = 255) String description,
    Map<String, String> metadata
) {
    public record PaymentMethodRequest(
        @NotBlank String type,
        @NotBlank String token
    ) {}
}

// ── Update request (partial) ──
public record UpdatePaymentRequest(
    @Size(max = 255) String description,
    Map<String, String> metadata
) {}
```

### Response Records

```java
// ── Single resource response ──
public record PaymentResponse(
    String id,
    String customerId,
    MoneyResponse amount,
    String status,
    String paymentType,
    FeeResponse fees,
    Instant createdAt,
    Instant updatedAt
) {
    public record MoneyResponse(String value, String currency) {}
    public record FeeResponse(String processing, String platform, String total) {}
}

// ── Collection response (paginated) ──
public record PagedResponse<T>(
    List<T> data,
    PaginationMeta pagination
) {
    public record PaginationMeta(
        int page,
        int size,
        long totalElements,
        int totalPages,
        boolean hasNext,
        boolean hasPrevious
    ) {}

    public static <T> PagedResponse<T> from(Page<T> page) {
        return new PagedResponse<>(
            page.getContent(),
            new PaginationMeta(
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages(),
                page.hasNext(),
                page.hasPrevious()
            )
        );
    }
}
```

### Async Operation Response (202 Accepted)

```java
// For long-running operations
public record AsyncOperationResponse(
    String operationId,
    String status,          // "PENDING", "PROCESSING", "COMPLETED", "FAILED"
    String statusUrl,       // URL to poll for status
    Instant estimatedCompletion
) {}

// Controller
@PostMapping("/payments/batch")
public ResponseEntity<AsyncOperationResponse> createBatch(
        @RequestBody BatchPaymentRequest request) {
    var operationId = batchService.submit(request);
    var response = new AsyncOperationResponse(
        operationId, "PENDING",
        "/api/v1/operations/" + operationId,
        Instant.now().plus(Duration.ofMinutes(5))
    );
    return ResponseEntity.accepted()
        .header("Location", response.statusUrl())
        .body(response);
}
```

---

## ❌ Pattern 4: Error Response Design

### Standard Error Response

```java
public record ApiError(
    String code,              // Machine-readable: "PAYMENT_DECLINED"
    String message,           // Human-readable: "Payment was declined by the issuer"
    String traceId,           // For support: "abc-123-def"
    Instant timestamp,
    String path,              // "/api/v1/payments"
    List<FieldError> errors   // Validation errors (optional)
) {
    public record FieldError(
        String field,         // "amount"
        String code,          // "POSITIVE"
        String message        // "must be greater than 0"
    ) {}
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ── Validation errors → 400 ──
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        var fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> new ApiError.FieldError(f.getField(), f.getCode(), f.getDefaultMessage()))
            .toList();

        return ResponseEntity.badRequest().body(new ApiError(
            "VALIDATION_FAILED",
            "Request validation failed",
            MDC.get("traceId"),
            Instant.now(),
            request.getRequestURI(),
            fieldErrors
        ));
    }

    // ── Resource not found → 404 ──
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ApiError(
            "RESOURCE_NOT_FOUND", ex.getMessage(),
            MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // ── Business rule violation → 422 ──
    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<ApiError> handleBusinessRule(
            BusinessRuleException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(new ApiError(
            ex.errorCode(), ex.getMessage(),
            MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // ── Duplicate / conflict → 409 ──
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiError> handleConflict(
            DuplicateResourceException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(new ApiError(
            "DUPLICATE_RESOURCE", ex.getMessage(),
            MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // ── Rate limited → 429 ──
    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<ApiError> handleRateLimit(
            RateLimitExceededException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", String.valueOf(ex.retryAfterSeconds()))
            .body(new ApiError(
                "RATE_LIMITED", "Too many requests. Retry after " + ex.retryAfterSeconds() + "s",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // ── Downstream failure → 502/503 ──
    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<ApiError> handleServiceDown(
            ServiceUnavailableException ex, HttpServletRequest request) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .header("Retry-After", "30")
            .body(new ApiError(
                "SERVICE_UNAVAILABLE", "Service temporarily unavailable",
                MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
    }

    // ── Catch-all → 500 ──
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception: {}", ex.getMessage(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(new ApiError(
            "INTERNAL_ERROR", "An unexpected error occurred",
            MDC.get("traceId"), Instant.now(), request.getRequestURI(), null));
        // ⚠️ Never expose ex.getMessage() to clients — could leak internals
    }
}
```

### Error Code Taxonomy

```
Format: <DOMAIN>_<CATEGORY>_<DETAIL>

Validation:
  VALIDATION_FAILED           — Generic validation failure
  FIELD_REQUIRED              — Missing required field
  FIELD_INVALID               — Field value is invalid

Payment:
  PAYMENT_DECLINED            — Issuer declined
  PAYMENT_INSUFFICIENT_FUNDS  — Not enough balance
  PAYMENT_DUPLICATE           — Duplicate idempotency key
  PAYMENT_EXPIRED             — Authorization expired
  PAYMENT_INVALID_STATE       — Invalid state transition

Authentication:
  AUTH_TOKEN_EXPIRED          — JWT expired
  AUTH_TOKEN_INVALID          — JWT malformed
  AUTH_INSUFFICIENT_SCOPE     — Missing required scope

System:
  RATE_LIMITED                — Too many requests
  SERVICE_UNAVAILABLE         — Downstream is down
  INTERNAL_ERROR              — Unhandled server error
```

---

## 🔑 Pattern 5: Idempotency

> **Rule:** Every mutating endpoint (POST, PATCH) should support idempotency for safe retries.

### Idempotency Key Pattern

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid CreatePaymentRequest request) {

    // Check for existing result
    var existing = idempotencyStore.find(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());  // Return cached result
    }

    // Process
    var result = paymentService.create(request, idempotencyKey);
    var response = mapper.toResponse(result);

    // Cache result
    idempotencyStore.save(idempotencyKey, response, Duration.ofHours(24));

    return ResponseEntity.status(HttpStatus.CREATED)
        .header("Location", "/api/v1/payments/" + result.id())
        .header("Idempotency-Key", idempotencyKey)
        .body(response);
}
```

### Natural Idempotency

```
GET    → Always idempotent (same result for same request)
PUT    → Naturally idempotent (full replace produces same result)
DELETE → Naturally idempotent (deleting already-deleted = 404 or 204)
POST   → NOT naturally idempotent → USE Idempotency-Key header
PATCH  → NOT naturally idempotent → USE If-Match (ETag) header
```

---

## 📖 Pattern 6: Pagination, Filtering & Sorting

### Pagination

```
GET /api/v1/payments?page=0&size=20&sort=createdAt,desc

Response includes pagination metadata:
{
  "data": [...],
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 1543,
    "totalPages": 78,
    "hasNext": true,
    "hasPrevious": false
  }
}
```

### Cursor-Based Pagination (For Large/Real-Time Datasets)

```
GET /api/v1/payments?limit=20&after=pay_abc123

Response:
{
  "data": [...],
  "cursors": {
    "after": "pay_xyz789",    ← Use as "after" for next page
    "before": "pay_abc124",   ← Use as "before" for previous page
    "hasMore": true
  }
}
```

### Filtering

```
GET /api/v1/payments?status=CAPTURED&customerId=cust-123&createdAfter=2026-01-01T00:00:00Z

Controller:
@GetMapping
public ResponseEntity<PagedResponse<PaymentResponse>> listPayments(
        @RequestParam(required = false) PaymentStatus status,
        @RequestParam(required = false) String customerId,
        @RequestParam(required = false) @DateTimeFormat(iso = ISO.DATE_TIME) Instant createdAfter,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") @Max(100) int size,
        @RequestParam(defaultValue = "createdAt,desc") String sort) {
    
    var criteria = new PaymentSearchCriteria(status, customerId, createdAfter);
    var pageable = PageRequest.of(page, size, parseSortParam(sort));
    var result = paymentQuery.search(criteria, pageable);
    return ResponseEntity.ok(PagedResponse.from(result.map(mapper::toResponse)));
}
```

---

## 🔄 Pattern 7: Versioning Strategy

### Comparison

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL path** | `/api/v1/payments` | Simple, explicit, cacheable | URL changes, breaks bookmarks |
| **Header** | `Accept: application/vnd.myapp.v2+json` | Clean URLs | Hard to test in browser |
| **Query param** | `/api/payments?version=2` | Easy to switch | Pollutes query string |

### Recommended: URL Path Versioning

```java
@RestController
@RequestMapping("/api/v1/payments")   // Version in path
public class PaymentControllerV1 {
    // v1 implementation
}

@RestController
@RequestMapping("/api/v2/payments")
public class PaymentControllerV2 {
    // v2 — can coexist with v1
}

// Evolution strategy:
// 1. Deploy v2 alongside v1
// 2. Migrate consumers to v2
// 3. Deprecate v1 (add Sunset header)
// 4. Remove v1 after migration period
```

### Deprecation Headers

```java
// Warn consumers about upcoming deprecation
@GetMapping("/api/v1/payments")
public ResponseEntity<...> listPayments() {
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Sat, 01 Jun 2026 00:00:00 GMT")
        .header("Link", "</api/v2/payments>; rel=\"successor-version\"")
        .body(result);
}
```

---

## 🔒 Pattern 8: Security Headers & CORS

```java
@Configuration
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentTypeOptions(Customizer.withDefaults())     // X-Content-Type-Options: nosniff
                .frameOptions(frame -> frame.deny())               // X-Frame-Options: DENY
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)))
            .cors(cors -> cors.configurationSource(corsConfig()))
            .csrf(csrf -> csrf.disable())          // Disable for API-only service
            .build();
    }

    private CorsConfigurationSource corsConfig() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "Idempotency-Key"));
        config.setExposedHeaders(List.of("Location", "Retry-After", "X-Trace-Id"));
        config.setMaxAge(3600L);

        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

---

## 📐 Pattern 9: Complete Controller Example

```java
@RestController
@RequestMapping("/api/v1/payments")
@Tag(name = "Payments", description = "Payment management endpoints")
public class PaymentController {

    private final ProcessPaymentUseCase processPayment;
    private final RefundPaymentUseCase refundPayment;
    private final PaymentQueryPort paymentQuery;
    private final PaymentApiMapper mapper;

    @PostMapping
    @Operation(summary = "Create a new payment")
    @ApiResponse(responseCode = "201", description = "Payment created")
    @ApiResponse(responseCode = "409", description = "Duplicate idempotency key")
    @ApiResponse(responseCode = "422", description = "Business rule violation")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid CreatePaymentRequest request) {

        var command = mapper.toCommand(request, idempotencyKey);
        var result = processPayment.process(command);

        return ResponseEntity.status(HttpStatus.CREATED)
            .header("Location", "/api/v1/payments/" + result.paymentId())
            .body(mapper.toResponse(result));
    }

    @GetMapping("/{paymentId}")
    @Operation(summary = "Get payment by ID")
    public ResponseEntity<PaymentResponse> getPayment(
            @PathVariable String paymentId) {
        return paymentQuery.findById(paymentId)
            .map(mapper::toResponse)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new ResourceNotFoundException("Payment", paymentId));
    }

    @GetMapping
    @Operation(summary = "Search payments")
    public ResponseEntity<PagedResponse<PaymentResponse>> searchPayments(
            @RequestParam(required = false) PaymentStatus status,
            @RequestParam(required = false) String customerId,
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size) {

        var criteria = new PaymentSearchCriteria(status, customerId, null, null);
        var pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "createdAt"));
        var result = paymentQuery.search(criteria, pageable);

        return ResponseEntity.ok(PagedResponse.from(result.map(mapper::toResponse)));
    }

    @PostMapping("/{paymentId}/refunds")
    @Operation(summary = "Refund a payment")
    public ResponseEntity<RefundResponse> refundPayment(
            @PathVariable String paymentId,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid RefundRequest request) {

        var command = mapper.toRefundCommand(paymentId, request, idempotencyKey);
        var result = refundPayment.refund(command);

        return ResponseEntity.status(HttpStatus.CREATED)
            .header("Location", "/api/v1/payments/" + paymentId + "/refunds/" + result.refundId())
            .body(mapper.toRefundResponse(result));
    }

    @DeleteMapping("/{paymentId}")
    @Operation(summary = "Cancel a pending payment")
    @ApiResponse(responseCode = "204", description = "Payment cancelled")
    public ResponseEntity<Void> cancelPayment(@PathVariable String paymentId) {
        paymentService.cancel(paymentId);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 🚫 REST API Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Verbs in URLs** | `/getPayments`, `/createPayment` | Use HTTP methods + nouns |
| **Returning 200 for errors** | Client can't distinguish success/failure | Use proper status codes |
| **Exposing internal errors** | Stack traces leak implementation details | Generic error messages + traceId |
| **No pagination** | Returns all records → OOM, slow | Always paginate collections |
| **Inconsistent error format** | Different shapes per endpoint | Single ApiError format everywhere |
| **No idempotency** | Retries create duplicates | Idempotency-Key header on POST |
| **Nested URLs > 2 levels** | `/a/1/b/2/c/3/d/4` — unreadable | Flatten or use query params |
| **Breaking changes without version** | Existing clients break | URL versioning + deprecation headers |
| **Returning nulls in JSON** | Client must null-check everything | Omit null fields (Jackson NON_NULL) |
| **No rate limiting** | DoS vulnerability | 429 + Retry-After |

---

## 💡 Golden Rules of REST API Design

```
1.  URLs are NOUNS (resources), HTTP methods are VERBS (actions).
2.  Status codes are your API's VOCABULARY — use them precisely.
3.  Error responses are a FEATURE — consistent, helpful, machine-readable.
4.  IDEMPOTENCY is mandatory for all mutating operations in distributed systems.
5.  PAGINATION is not optional — unbounded responses are production incidents.
6.  VERSION from day one — it's free now, expensive to add later.
7.  Validate ALL input — never trust the client, even internal clients.
8.  Return ONLY what consumers need — over-fetching wastes bandwidth and leaks data.
9.  traceId in every error response — it's the bridge between API consumers and your logs.
10. Design for the CONSUMER, not the database — API shape ≠ database schema.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Spring Web / Jackson / OpenAPI*
