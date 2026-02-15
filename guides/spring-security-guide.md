# Effective Modern Spring Security

A comprehensive guide to securing Spring Boot applications with OAuth2, JWT, method security, and production-ready patterns.

---

## Table of Contents

1. [Security Fundamentals](#security-fundamentals)
2. [Spring Security Architecture](#spring-security-architecture)
3. [Authentication Basics](#authentication-basics)
4. [Authorization Patterns](#authorization-patterns)
5. [OAuth2 & OpenID Connect](#oauth2--openid-connect)
6. [JWT Token Management](#jwt-token-management)
7. [Method Security](#method-security)
8. [CORS & CSRF Protection](#cors--csrf-protection)
9. [Password Management](#password-management)
10. [Session Management](#session-management)
11. [Security Headers](#security-headers)
12. [Multi-Tenancy Security](#multi-tenancy-security)
13. [API Key Authentication](#api-key-authentication)
14. [Rate Limiting & Throttling](#rate-limiting--throttling)
15. [Security Testing](#security-testing)
16. [Common Security Vulnerabilities](#common-security-vulnerabilities)
17. [Production Security Checklist](#production-security-checklist)

---

## Security Fundamentals

### Authentication vs Authorization

```
Authentication: WHO are you?
Authorization: WHAT can you do?

┌─────────────────────────────────────────────────────┐
│                 Security Flow                        │
├─────────────────────────────────────────────────────┤
│  1. Authentication → Verify Identity                │
│     - Username/Password                             │
│     - OAuth2/OIDC                                   │
│     - API Keys                                      │
│     - Certificates                                  │
│                                                     │
│  2. Authorization → Check Permissions               │
│     - Role-based (RBAC)                            │
│     - Attribute-based (ABAC)                       │
│     - Resource-based                               │
│                                                     │
│  3. Access Control → Enforce Rules                 │
│     - Method security                              │
│     - URL patterns                                 │
│     - Data filtering                               │
└─────────────────────────────────────────────────────┘
```

### Security Principles

1. **Defense in Depth** — Multiple layers of security
2. **Least Privilege** — Minimum necessary permissions
3. **Fail Secure** — Deny by default
4. **Complete Mediation** — Check every access
5. **Open Design** — Security through design, not obscurity

---

## Spring Security Architecture

### Core Components

```java
// SecurityFilterChain - The heart of Spring Security
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf.disable()) // For stateless APIs
            .build();
    }
}
```

### Filter Chain Order

```
1. SecurityContextPersistenceFilter
2. LogoutFilter
3. UsernamePasswordAuthenticationFilter
4. BasicAuthenticationFilter
5. BearerTokenAuthenticationFilter (OAuth2)
6. ExceptionTranslationFilter
7. FilterSecurityInterceptor
```

---

## Authentication Basics

### In-Memory Authentication (Development Only)

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder().encode("password"))
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder().encode("admin"))
        .roles("USER", "ADMIN")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### Database Authentication

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .toList())
            .accountExpired(!user.isActive())
            .accountLocked(user.isLocked())
            .credentialsExpired(user.isPasswordExpired())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

---

## OAuth2 & OpenID Connect

### OAuth2 Resource Server (JWT)

```java
@Configuration
public class OAuth2ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http)
            throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtAuthenticationConverter =
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);

        return jwtAuthenticationConverter;
    }
}
```

### Application Properties

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-auth-server.com
          # OR
          jwk-set-uri: https://your-auth-server.com/.well-known/jwks.json

      client:
        registration:
          keycloak:
            client-id: my-app
            client-secret: ${OAUTH2_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/my-realm
```

---

## JWT Token Management

### Custom JWT Tokens

```java
@Service
public class JwtTokenService {

    private final JwtEncoder jwtEncoder;
    private final JwtDecoder jwtDecoder;

    public String generateToken(Authentication authentication) {
        Instant now = Instant.now();
        long expiry = 3600; // 1 hour

        String scope = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.joining(" "));

        JwtClaimsSet claims = JwtClaimsSet.builder()
            .issuer("self")
            .issuedAt(now)
            .expiresAt(now.plusSeconds(expiry))
            .subject(authentication.getName())
            .claim("scope", scope)
            .claim("email", getUserEmail(authentication))
            .claim("tenant", getUserTenant(authentication))
            .build();

        return jwtEncoder.encode(JwtEncoderParameters.from(claims))
            .getTokenValue();
    }

    public boolean validateToken(String token) {
        try {
            Jwt jwt = jwtDecoder.decode(token);
            return jwt.getExpiresAt().isAfter(Instant.now());
        } catch (JwtException e) {
            return false;
        }
    }
}
```

### JWT Configuration

```java
@Configuration
public class JwtConfig {

    @Value("${jwt.public.key}")
    private RSAPublicKey publicKey;

    @Value("${jwt.private.key}")
    private RSAPrivateKey privateKey;

    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withPublicKey(publicKey).build();
    }

    @Bean
    public JwtEncoder jwtEncoder() {
        JWK jwk = new RSAKey.Builder(publicKey)
            .privateKey(privateKey)
            .build();
        JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(
            new JWKSet(jwk));
        return new NimbusJwtEncoder(jwks);
    }
}
```

---

## Method Security

### Enable Method Security

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,
    securedEnabled = true,
    jsr250Enabled = true
)
public class MethodSecurityConfig {
}
```

### Using Annotations

```java
@Service
public class PaymentService {

    // Pre-authorization
    @PreAuthorize("hasRole('ADMIN')")
    public void deletePayment(String paymentId) {
        // Only admins can delete
    }

    // Check parameter
    @PreAuthorize("#userId == authentication.principal.id")
    public Payment getUserPayment(String userId, String paymentId) {
        // Users can only access their own payments
    }

    // Post-authorization - check return value
    @PostAuthorize("returnObject.userId == authentication.principal.id")
    public Payment getPayment(String paymentId) {
        return paymentRepository.findById(paymentId);
    }

    // Filter collection before returning
    @PostFilter("filterObject.userId == authentication.principal.id")
    public List<Payment> getAllPayments() {
        return paymentRepository.findAll();
    }

    // SpEL with custom method
    @PreAuthorize("@paymentSecurity.canAccess(authentication, #paymentId)")
    public Payment processPayment(String paymentId) {
        // Custom security logic
    }
}
```

### Custom Security Expressions

```java
@Component("paymentSecurity")
public class PaymentSecurityExpressions {

    private final PaymentRepository paymentRepository;

    public boolean canAccess(Authentication authentication, String paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
            .orElse(null);

        if (payment == null) {
            return false;
        }

        String username = authentication.getName();

        // Admin can access all
        if (hasRole(authentication, "ADMIN")) {
            return true;
        }

        // Owner can access
        if (payment.getUserId().equals(username)) {
            return true;
        }

        // Merchant can access their payments
        return hasRole(authentication, "MERCHANT") &&
               payment.getMerchantId().equals(username);
    }

    private boolean hasRole(Authentication auth, String role) {
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }
}
```

---

## CORS & CSRF Protection

### CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList(
            "https://app.example.com",
            "https://admin.example.com"
        ));
        configuration.setAllowedMethods(Arrays.asList(
            "GET", "POST", "PUT", "DELETE", "PATCH"
        ));
        configuration.setAllowedHeaders(Arrays.asList(
            "Authorization",
            "Content-Type",
            "X-Request-ID"
        ));
        configuration.setExposedHeaders(Arrays.asList(
            "X-Total-Count",
            "X-Page-Number"
        ));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source =
            new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);

        return source;
    }
}
```

### CSRF Protection

```java
// For stateless APIs - disable CSRF
@Bean
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .build();
}

// For traditional web apps - enable CSRF
@Bean
public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        )
        .build();
}
```

---

## Production Security Checklist

### Must-Have Security Measures

- [ ] **HTTPS Only** — Enforce TLS in production
- [ ] **Password Encryption** — Use BCrypt (never plaintext)
- [ ] **JWT Validation** — Verify signature, expiration, issuer
- [ ] **Rate Limiting** — Prevent brute force attacks
- [ ] **CORS Configuration** — Restrict allowed origins
- [ ] **Security Headers** — CSP, X-Frame-Options, etc.
- [ ] **Input Validation** — Sanitize all user input
- [ ] **SQL Injection Prevention** — Use parameterized queries
- [ ] **XSS Prevention** — Escape output, use CSP
- [ ] **Dependency Scanning** — Check for vulnerable libraries
- [ ] **Secrets Management** — Use vault, not hardcoded
- [ ] **Audit Logging** — Log authentication events
- [ ] **Session Timeout** — Auto-logout inactive users
- [ ] **MFA Support** — Multi-factor authentication
- [ ] **Role-Based Access** — Principle of least privilege

### Security Headers

```java
@Bean
public SecurityFilterChain securityHeaders(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; " +
                    "script-src 'self' 'unsafe-inline'; " +
                    "style-src 'self' 'unsafe-inline'")
            )
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.headerValue(
                XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK
            ))
            .contentTypeOptions(Customizer.withDefaults())
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy
                    .STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
            )
            .permissionsPolicy(permissions -> permissions
                .policy("geolocation=(), camera=(), microphone=()")
            )
        )
        .build();
}
```

---

## Best Practices

### ✅ DO

- Use BCrypt or Argon2 for password hashing
- Implement JWT token rotation
- Use HTTPS in production
- Enable CORS only for trusted origins
- Implement rate limiting on authentication endpoints
- Log security events (login, logout, failed attempts)
- Use method security for fine-grained control
- Validate and sanitize all inputs
- Implement proper session management
- Use OAuth2/OIDC for modern applications

### ❌ DON'T

- Store passwords in plaintext
- Use MD5 or SHA-1 for passwords
- Hardcode secrets in source code
- Disable CSRF without good reason
- Trust client-side validation alone
- Use overly broad CORS configuration
- Expose stack traces in production
- Use default credentials
- Ignore security updates
- Store sensitive data in JWT claims

---

## Common Pitfalls

### 1. **Stateless JWT + Database Calls**

```java
// ❌ BAD: Defeating the purpose of stateless JWT
@PreAuthorize("#userId == authentication.principal.id")
public Payment getPayment(String userId) {
    User user = userRepository.findById(userId); // DB call on every request!
}

// ✅ GOOD: Embed necessary claims in JWT
public String generateToken(User user) {
    return JWT.create()
        .withClaim("userId", user.getId())
        .withClaim("roles", user.getRoles())
        .sign(algorithm);
}
```

### 2. **Improper Password Storage**

```java
// ❌ BAD: Weak hashing
String hashedPassword = DigestUtils.md5Hex(password);

// ✅ GOOD: Strong adaptive hashing
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hashedPassword = encoder.encode(password);
```

---

*This guide provides a comprehensive foundation for implementing Spring Security in production environments. Always perform security audits and penetration testing before deploying to production.*
