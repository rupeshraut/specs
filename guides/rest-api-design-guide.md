# REST API Design & Implementation Complete Guide

A comprehensive guide to designing production-grade REST APIs with Spring Boot, covering design principles, implementation patterns, versioning, and best practices.

---

## Table of Contents

1. [REST Fundamentals](#rest-fundamentals)
2. [Resource Design](#resource-design)
3. [HTTP Methods](#http-methods)
4. [Status Codes](#status-codes)
5. [Request/Response Design](#requestresponse-design)
6. [Error Handling](#error-handling)
7. [Pagination](#pagination)
8. [Filtering & Sorting](#filtering--sorting)
9. [Versioning Strategies](#versioning-strategies)
10. [HATEOAS](#hateoas)
11. [API Documentation](#api-documentation)
12. [Security](#security)
13. [Rate Limiting](#rate-limiting)
14. [Caching](#caching)
15. [Production Patterns](#production-patterns)

---

## REST Fundamentals

### REST Principles

```
1. Client-Server
   - Separation of concerns
   - Independent evolution

2. Stateless
   - No session state on server
   - Each request self-contained

3. Cacheable
   - Responses explicitly cacheable/non-cacheable

4. Uniform Interface
   - Resources identified by URIs
   - Standard HTTP methods
   - Self-descriptive messages

5. Layered System
   - Client unaware of intermediaries

6. Code on Demand (optional)
   - Server can extend client functionality
```

### RESTful Maturity Model

```
Level 0: Single URI, Single Method (RPC)
  POST /api/service

Level 1: Multiple URIs, Single Method
  POST /api/users
  POST /api/payments

Level 2: HTTP Verbs (Most REST APIs)
  GET    /api/users
  POST   /api/users
  PUT    /api/users/123
  DELETE /api/users/123

Level 3: HATEOAS (Hypermedia)
  GET /api/users/123
  Response includes links to related resources
```

---

## Resource Design

### URI Design

```java
// ✅ GOOD: Nouns, plural, hierarchical
GET    /api/users
GET    /api/users/123
GET    /api/users/123/orders
GET    /api/orders/456/items

// ❌ BAD: Verbs, actions in URI
GET    /api/getUser?id=123
POST   /api/createUser
GET    /api/deleteUser/123

// ✅ GOOD: Query parameters for filtering
GET    /api/users?status=active&role=admin

// ❌ BAD: Actions as resources
POST   /api/users/123/activate
POST   /api/orders/456/cancel

// ✅ GOOD: Use HTTP methods or sub-resources
PUT    /api/users/123 { "status": "active" }
POST   /api/orders/456/cancellations
```

### Resource Naming

```
Rules:
1. Use nouns, not verbs
2. Use plural for collections
3. Use lowercase
4. Use hyphens for multi-word
5. No trailing slashes
6. Keep it simple and consistent

Examples:
✅ /api/payment-methods
✅ /api/users/123/addresses
✅ /api/orders/456/line-items

❌ /api/paymentMethods
❌ /api/user/123/address
❌ /api/orders/456/lineItems/
```

---

## HTTP Methods

### Standard Methods

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET - Retrieve resource(s)
    @GetMapping
    public ResponseEntity<Page<UserResponse>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        Page<UserResponse> users = userService.findAll(
            PageRequest.of(page, size)
        );

        return ResponseEntity.ok(users);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable String id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST - Create new resource
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request,
            UriComponentsBuilder uriBuilder) {

        UserResponse created = userService.create(request);

        URI location = uriBuilder
            .path("/api/users/{id}")
            .buildAndExpand(created.getId())
            .toUri();

        return ResponseEntity
            .created(location)
            .body(created);
    }

    // PUT - Replace entire resource
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable String id,
            @Valid @RequestBody UpdateUserRequest request) {

        UserResponse updated = userService.update(id, request);
        return ResponseEntity.ok(updated);
    }

    // PATCH - Partial update
    @PatchMapping("/{id}")
    public ResponseEntity<UserResponse> patchUser(
            @PathVariable String id,
            @Valid @RequestBody PatchUserRequest request) {

        UserResponse patched = userService.patch(id, request);
        return ResponseEntity.ok(patched);
    }

    // DELETE - Remove resource
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable String id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Idempotency

```
Method    Idempotent?  Safe?
GET       Yes          Yes
PUT       Yes          No
PATCH     No*          No
DELETE    Yes          No
POST      No           No

*PATCH can be idempotent depending on implementation

Idempotent: Same request can be made multiple times
            with same result

Safe: Read-only, doesn't modify state
```

---

## Status Codes

### Common Status Codes

```java
// 2xx Success
200 OK              - GET, PUT, PATCH successful
201 Created         - POST successful (include Location header)
202 Accepted        - Async processing started
204 No Content      - DELETE successful, no body

// 3xx Redirection
301 Moved Permanently
302 Found
304 Not Modified    - Conditional GET, resource unchanged

// 4xx Client Errors
400 Bad Request     - Invalid syntax, validation failed
401 Unauthorized    - Authentication required
403 Forbidden       - Authenticated but not authorized
404 Not Found       - Resource doesn't exist
405 Method Not Allowed
409 Conflict        - Concurrent modification
422 Unprocessable Entity - Semantic validation failed
429 Too Many Requests - Rate limit exceeded

// 5xx Server Errors
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

### Implementation

```java
@RestController
public class PaymentController {

    @PostMapping("/api/payments")
    public ResponseEntity<?> createPayment(
            @Valid @RequestBody PaymentRequest request) {

        try {
            Payment payment = paymentService.create(request);

            return ResponseEntity
                .status(HttpStatus.CREATED)
                .header("Location", "/api/payments/" + payment.getId())
                .body(payment);

        } catch (InsufficientFundsException e) {
            return ResponseEntity
                .status(HttpStatus.UNPROCESSABLE_ENTITY)
                .body(ErrorResponse.of("INSUFFICIENT_FUNDS", e.getMessage()));

        } catch (DuplicatePaymentException e) {
            return ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body(ErrorResponse.of("DUPLICATE_PAYMENT", e.getMessage()));
        }
    }

    @GetMapping("/api/payments/{id}")
    public ResponseEntity<Payment> getPayment(@PathVariable String id) {
        return paymentService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/api/payments/{id}")
    public ResponseEntity<Void> deletePayment(@PathVariable String id) {
        if (!paymentService.exists(id)) {
            return ResponseEntity.notFound().build();
        }

        paymentService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Request/Response Design

### Request DTOs

```java
// Create request - only fields needed for creation
public record CreatePaymentRequest(
    @NotNull @Min(1)
    BigDecimal amount,

    @NotBlank @Size(min = 3, max = 3)
    String currency,

    @NotBlank
    String userId,

    @Valid
    PaymentMethodDto paymentMethod
) {}

// Update request - all updatable fields
public record UpdatePaymentRequest(
    @NotBlank
    String status,

    String description
) {}

// Patch request - optional fields (null = no change)
public record PatchPaymentRequest(
    String status,
    String description,
    Map<String, String> metadata
) {}
```

### Response DTOs

```java
// Response with links (HATEOAS)
public record PaymentResponse(
    String id,
    BigDecimal amount,
    String currency,
    String status,
    Instant createdAt,
    Instant updatedAt,
    Map<String, Link> _links
) {
    public static PaymentResponse from(Payment payment) {
        return new PaymentResponse(
            payment.getId(),
            payment.getAmount(),
            payment.getCurrency(),
            payment.getStatus().name(),
            payment.getCreatedAt(),
            payment.getUpdatedAt(),
            createLinks(payment)
        );
    }

    private static Map<String, Link> createLinks(Payment payment) {
        return Map.of(
            "self", Link.of("/api/payments/" + payment.getId()),
            "cancel", Link.of("/api/payments/" + payment.getId() + "/cancel"),
            "user", Link.of("/api/users/" + payment.getUserId())
        );
    }
}

public record Link(String href, String method) {
    public static Link of(String href) {
        return new Link(href, "GET");
    }

    public static Link of(String href, String method) {
        return new Link(href, method);
    }
}
```

---

## Error Handling

### Standard Error Response

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(404)
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .traceId(MDC.get("traceId"))
            .build();

        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {

        Map<String, String> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> error.getDefaultMessage() != null ?
                    error.getDefaultMessage() : "Invalid value"
            ));

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(400)
            .error("Validation Failed")
            .message("Invalid request parameters")
            .path(request.getRequestURI())
            .traceId(MDC.get("traceId"))
            .errors(fieldErrors)
            .build();

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(error);
    }
}

@Builder
public record ErrorResponse(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path,
    String traceId,
    Map<String, String> errors
) {}
```

---

## Pagination

### Offset-Based Pagination

```java
@GetMapping("/api/users")
public ResponseEntity<PageResponse<UserResponse>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt,desc") String[] sort) {

    Page<User> users = userService.findAll(
        PageRequest.of(page, size, Sort.by(parseSort(sort)))
    );

    PageResponse<UserResponse> response = PageResponse.of(
        users.map(UserResponse::from),
        "/api/users"
    );

    return ResponseEntity.ok(response);
}

public record PageResponse<T>(
    List<T> content,
    PageMetadata metadata,
    Map<String, String> links
) {
    public static <T> PageResponse<T> of(Page<T> page, String baseUrl) {
        return new PageResponse<>(
            page.getContent(),
            new PageMetadata(
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages()
            ),
            createLinks(page, baseUrl)
        );
    }

    private static <T> Map<String, String> createLinks(Page<T> page, String baseUrl) {
        Map<String, String> links = new HashMap<>();

        links.put("self", baseUrl + "?page=" + page.getNumber() +
                          "&size=" + page.getSize());

        if (page.hasNext()) {
            links.put("next", baseUrl + "?page=" + (page.getNumber() + 1) +
                              "&size=" + page.getSize());
        }

        if (page.hasPrevious()) {
            links.put("prev", baseUrl + "?page=" + (page.getNumber() - 1) +
                              "&size=" + page.getSize());
        }

        links.put("first", baseUrl + "?page=0&size=" + page.getSize());
        links.put("last", baseUrl + "?page=" + (page.getTotalPages() - 1) +
                         "&size=" + page.getSize());

        return links;
    }
}

public record PageMetadata(
    int page,
    int size,
    long totalElements,
    int totalPages
) {}
```

### Cursor-Based Pagination

```java
// Better for large datasets
@GetMapping("/api/payments")
public ResponseEntity<CursorPageResponse<PaymentResponse>> getPayments(
        @RequestParam(required = false) String cursor,
        @RequestParam(defaultValue = "20") int size) {

    CursorPage<Payment> page = paymentService.findAll(cursor, size);

    CursorPageResponse<PaymentResponse> response = CursorPageResponse.of(
        page.getContent().stream()
            .map(PaymentResponse::from)
            .toList(),
        page.getNextCursor(),
        page.hasMore()
    );

    return ResponseEntity.ok(response);
}

public record CursorPageResponse<T>(
    List<T> content,
    String nextCursor,
    boolean hasMore
) {
    public static <T> CursorPageResponse<T> of(
            List<T> content,
            String nextCursor,
            boolean hasMore) {
        return new CursorPageResponse<>(content, nextCursor, hasMore);
    }
}
```

---

## Versioning Strategies

### URI Versioning

```java
// Most common, explicit
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // v1 implementation
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // v2 implementation with breaking changes
}
```

### Header Versioning

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(headers = "API-Version=1")
    public ResponseEntity<UserV1Response> getUserV1(@PathVariable String id) {
        // V1 implementation
    }

    @GetMapping(headers = "API-Version=2")
    public ResponseEntity<UserV2Response> getUserV2(@PathVariable String id) {
        // V2 implementation
    }
}
```

### Content Negotiation

```java
@GetMapping(
    value = "/{id}",
    produces = "application/vnd.company.user.v1+json"
)
public ResponseEntity<UserV1Response> getUserV1(@PathVariable String id) {
    // V1 implementation
}

@GetMapping(
    value = "/{id}",
    produces = "application/vnd.company.user.v2+json"
)
public ResponseEntity<UserV2Response> getUserV2(@PathVariable String id) {
    // V2 implementation
}

// Request:
// Accept: application/vnd.company.user.v2+json
```

---

## API Documentation

### OpenAPI/Swagger

```java
@Configuration
public class OpenAPIConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Payment API")
                .version("1.0")
                .description("Payment processing API")
                .contact(new Contact()
                    .name("API Support")
                    .email("api@company.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .servers(List.of(
                new Server()
                    .url("https://api.company.com")
                    .description("Production"),
                new Server()
                    .url("https://api-staging.company.com")
                    .description("Staging")
            ));
    }
}

@RestController
@RequestMapping("/api/payments")
@Tag(name = "Payments", description = "Payment operations")
public class PaymentController {

    @Operation(
        summary = "Create payment",
        description = "Creates a new payment for the user"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "Payment created successfully",
            content = @Content(schema = @Schema(implementation = PaymentResponse.class))
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Invalid request",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "422",
            description = "Insufficient funds",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @Parameter(description = "Payment details")
            @Valid @RequestBody CreatePaymentRequest request) {
        // Implementation
    }
}
```

---

## Best Practices

### ✅ DO

- Use nouns for resources, not verbs
- Use plural names for collections
- Use HTTP methods correctly
- Return appropriate status codes
- Include pagination for collections
- Version your API from day one
- Document with OpenAPI/Swagger
- Use DTOs, not domain entities
- Validate all inputs
- Include correlation IDs

### ❌ DON'T

- Use verbs in URIs
- Expose database IDs directly
- Return 200 for all responses
- Skip error handling
- Return unbounded collections
- Break existing API contracts
- Skip API documentation
- Expose internal models
- Trust client input
- Log sensitive data

---

*This guide provides comprehensive patterns for designing production-grade REST APIs. Good API design is an investment that pays dividends in developer experience and system maintainability.*
