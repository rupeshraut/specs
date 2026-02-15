# API Gateway & Service Mesh Patterns

A comprehensive guide to implementing API gateways and service mesh for microservices architectures with Spring Cloud Gateway, Istio patterns, and production operations.

---

## Table of Contents

1. [Gateway vs Service Mesh](#gateway-vs-service-mesh)
2. [API Gateway Patterns](#api-gateway-patterns)
3. [Spring Cloud Gateway](#spring-cloud-gateway)
4. [Gateway Routing](#gateway-routing)
5. [Authentication & Authorization](#authentication--authorization)
6. [Rate Limiting](#rate-limiting)
7. [Circuit Breaking](#circuit-breaking)
8. [Request/Response Transformation](#requestresponse-transformation)
9. [Service Mesh Fundamentals](#service-mesh-fundamentals)
10. [Istio Patterns](#istio-patterns)
11. [Traffic Management](#traffic-management)
12. [Security with mTLS](#security-with-mtls)
13. [Observability](#observability)
14. [Production Patterns](#production-patterns)

---

## Gateway vs Service Mesh

### Comparison

```
┌─────────────────────────────────────────────────────────┐
│                   API Gateway                            │
│  - External traffic entry point                         │
│  - Authentication/Authorization                          │
│  - Rate limiting                                         │
│  - Request routing                                       │
│  - Protocol translation (REST → gRPC)                   │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              Service Mesh (Istio/Linkerd)               │
│  - Service-to-service communication                     │
│  - mTLS encryption                                       │
│  - Load balancing                                        │
│  - Circuit breaking                                      │
│  - Traffic splitting (canary, A/B)                      │
│  - Distributed tracing                                   │
└─────────────────────────────────────────────────────────┘

Use Both:
- Gateway: North-South traffic (external → internal)
- Service Mesh: East-West traffic (service → service)
```

### When to Use What

```yaml
API Gateway:
  Use When:
    - Need single entry point for clients
    - External authentication required
    - API composition needed
    - Protocol translation required
    - Rate limiting per client

  Examples:
    - Mobile app → Backend services
    - Third-party integrations
    - Public API exposure

Service Mesh:
  Use When:
    - Many microservices (10+)
    - Service-to-service encryption needed
    - Complex traffic routing required
    - Observability across services
    - Zero-trust networking

  Examples:
    - Payment service → Order service
    - Canary deployments
    - Circuit breaking between services
```

---

## API Gateway Patterns

### Gateway Aggregation

```java
// Combine multiple backend calls into one response
@RestController
@RequestMapping("/api/gateway")
public class AggregationController {

    private final WebClient userService;
    private final WebClient orderService;
    private final WebClient paymentService;

    @GetMapping("/user-dashboard/{userId}")
    public Mono<UserDashboard> getUserDashboard(@PathVariable String userId) {

        Mono<UserProfile> userProfile = userService
            .get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserProfile.class);

        Mono<List<Order>> orders = orderService
            .get()
            .uri("/orders?userId={userId}", userId)
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList();

        Mono<PaymentSummary> payments = paymentService
            .get()
            .uri("/payments/summary?userId={userId}", userId)
            .retrieve()
            .bodyToMono(PaymentSummary.class);

        // Combine all results
        return Mono.zip(userProfile, orders, payments)
            .map(tuple -> new UserDashboard(
                tuple.getT1(), // user
                tuple.getT2(), // orders
                tuple.getT3()  // payments
            ));
    }
}
```

---

## Spring Cloud Gateway

### Configuration

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # User Service Route
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: userServiceCircuitBreaker
                fallbackUri: forward:/fallback/users

        # Payment Service Route
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

      # Global CORS
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "https://app.example.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
            allowedHeaders: "*"
            maxAge: 3600
```

### Custom Filters

```java
@Component
public class AuthenticationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<AuthenticationGatewayFilterFactory.Config> {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            String authHeader = request.getHeaders().getFirst("Authorization");

            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            String token = authHeader.substring(7);

            try {
                Claims claims = jwtTokenProvider.validateToken(token);

                ServerHttpRequest mutatedRequest = request.mutate()
                    .header("X-User-ID", claims.getSubject())
                    .header("X-User-Roles", claims.get("roles", String.class))
                    .build();

                return chain.filter(
                    exchange.mutate().request(mutatedRequest).build()
                );

            } catch (Exception e) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
        };
    }

    public static class Config {}
}
```

---

## Rate Limiting

### Redis-Based Rate Limiter

```java
@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-ID");

            return Mono.justOrEmpty(userId)
                .defaultIfEmpty("anonymous");
        };
    }
}
```

```yaml
# Route-specific rate limiting
spring:
  cloud:
    gateway:
      routes:
        - id: payment-service
          uri: lb://payment-service
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
```

---

## Service Mesh Fundamentals

### Istio Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 Control Plane (istiod)                   │
│  - Pilot: Service discovery, traffic management         │
│  - Citadel: Certificate management, mTLS               │
└────────────────────┬────────────────────────────────────┘
                     │ Configuration
        ┌────────────┴────────────┐
        ▼                         ▼
┌───────────────┐         ┌───────────────┐
│  Service A    │         │  Service B    │
│ ┌───────────┐ │  mTLS   │ ┌───────────┐ │
│ │ Envoy     │◄├─────────┤►│ Envoy     │ │
│ │ Sidecar   │ │         │ │ Sidecar   │ │
│ └───────────┘ │         │ └───────────┘ │
└───────────────┘         └───────────────┘
```

---

## Traffic Management

### Virtual Service

```yaml
# Canary deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: stable
          weight: 90
        - destination:
            host: payment-service
            subset: canary
          weight: 10
```

### Destination Rule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service

  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST

    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s

  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

---

## Security with mTLS

### Strict mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Authorization Policy

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-authz
spec:
  selector:
    matchLabels:
      app: payment-service

  action: ALLOW

  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/order-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments"]
```

---

## Production Best Practices

### ✅ DO

- Use API Gateway for external traffic
- Implement service mesh for microservices
- Enable mTLS in production
- Set up proper rate limiting
- Use circuit breakers
- Implement request timeouts
- Monitor gateway metrics
- Version your APIs
- Use health checks
- Implement distributed tracing

### ❌ DON'T

- Expose internal services directly
- Skip authentication at gateway
- Ignore rate limits
- Use default timeouts
- Skip circuit breaking
- Forget about observability
- Hardcode service URLs
- Skip mTLS in production

---

*This guide provides comprehensive patterns for API gateways and service mesh. Choose the right tool: gateways for north-south traffic, service mesh for east-west traffic.*
