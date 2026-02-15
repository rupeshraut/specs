# Application Security Guide for Java

A comprehensive guide to building secure Java applications covering OWASP Top 10, secure coding practices, dependency management, and production security patterns.

---

## Table of Contents

1. [Security Fundamentals](#security-fundamentals)
2. [OWASP Top 10](#owasp-top-10)
3. [Injection Prevention](#injection-prevention)
4. [Authentication Security](#authentication-security)
5. [Authorization Patterns](#authorization-patterns)
6. [Cryptography](#cryptography)
7. [Dependency Security](#dependency-security)
8. [Secure Configuration](#secure-configuration)
9. [API Security](#api-security)
10. [Security Testing](#security-testing)
11. [Production Hardening](#production-hardening)

---

## Security Fundamentals

### Defense in Depth

```
Security Layers:

1. Network Layer
   - Firewalls
   - VPN/Private networks
   - DDoS protection

2. Infrastructure Layer
   - Container security
   - Kubernetes RBAC
   - Network policies

3. Application Layer
   - Input validation
   - Authentication/Authorization
   - Secure coding

4. Data Layer
   - Encryption at rest
   - Encryption in transit
   - Data masking

Each layer provides protection even if others fail
```

### Security Principles

```
1. Least Privilege
   - Grant minimum necessary permissions
   - Time-limited access
   - Regular access reviews

2. Defense in Depth
   - Multiple security layers
   - Fail securely
   - Assume breach

3. Zero Trust
   - Verify explicitly
   - Never trust, always verify
   - Assume compromised network

4. Fail Securely
   - Default deny
   - Graceful degradation
   - Secure error messages

5. Keep it Simple
   - Complex systems are insecure
   - Minimize attack surface
   - Avoid security through obscurity
```

---

## OWASP Top 10

### A01: Broken Access Control

```java
// ❌ BAD: No authorization check
@GetMapping("/api/users/{userId}/orders")
public List<Order> getOrders(@PathVariable String userId) {
    return orderService.getOrdersByUserId(userId);
    // Any user can access any other user's orders!
}

// ✅ GOOD: Verify ownership
@GetMapping("/api/users/{userId}/orders")
public List<Order> getOrders(@PathVariable String userId,
                             @AuthenticationPrincipal User currentUser) {

    if (!currentUser.getId().equals(userId) && !currentUser.isAdmin()) {
        throw new AccessDeniedException("Cannot access other user's orders");
    }

    return orderService.getOrdersByUserId(userId);
}

// ✅ BETTER: Use @PreAuthorize
@GetMapping("/api/users/{userId}/orders")
@PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
public List<Order> getOrders(@PathVariable String userId) {
    return orderService.getOrdersByUserId(userId);
}
```

### A02: Cryptographic Failures

```java
// ❌ BAD: Storing plaintext passwords
@Entity
public class User {
    private String password; // Never store plaintext!
}

// ✅ GOOD: Hash passwords with strong algorithm
@Service
public class UserService {

    private final PasswordEncoder passwordEncoder;

    public User createUser(String username, String password) {
        // Use BCrypt with work factor 12+
        String hashedPassword = passwordEncoder.encode(password);

        return User.builder()
            .username(username)
            .password(hashedPassword)
            .build();
    }

    public boolean authenticate(String username, String password) {
        User user = userRepository.findByUsername(username)
            .orElseThrow();

        // Constant-time comparison
        return passwordEncoder.matches(password, user.getPassword());
    }
}

// Spring Security configuration
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // Work factor 12
}
```

### A03: Injection

```java
// ❌ BAD: SQL Injection vulnerability
@Repository
public class UserRepository {
    public User findByUsername(String username) {
        String sql = "SELECT * FROM users WHERE username = '" + username + "'";
        // username = "admin' OR '1'='1" → SQLi attack!
        return jdbcTemplate.queryForObject(sql, userMapper);
    }
}

// ✅ GOOD: Parameterized queries
@Repository
public class UserRepository {
    public User findByUsername(String username) {
        String sql = "SELECT * FROM users WHERE username = ?";
        return jdbcTemplate.queryForObject(sql, userMapper, username);
    }
}

// ✅ BETTER: Use Spring Data JPA
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}

// ❌ BAD: Command injection
public void backupDatabase(String filename) {
    String command = "pg_dump -f " + filename;
    Runtime.getRuntime().exec(command); // Command injection!
}

// ✅ GOOD: Use array form
public void backupDatabase(String filename) {
    // Validate filename first
    if (!filename.matches("[a-zA-Z0-9_-]+\\.sql")) {
        throw new IllegalArgumentException("Invalid filename");
    }

    String[] command = {"pg_dump", "-f", filename};
    Runtime.getRuntime().exec(command);
}
```

---

## Injection Prevention

### Input Validation

```java
@RestController
@Validated
public class PaymentController {

    @PostMapping("/api/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @Valid @RequestBody CreatePaymentRequest request) {

        return ResponseEntity.ok(paymentService.createPayment(request));
    }
}

// Request DTO with validation
public record CreatePaymentRequest(

    @NotNull(message = "Amount is required")
    @Positive(message = "Amount must be positive")
    @DecimalMax(value = "1000000", message = "Amount too large")
    BigDecimal amount,

    @NotBlank(message = "Currency is required")
    @Pattern(regexp = "^[A-Z]{3}$", message = "Invalid currency code")
    String currency,

    @NotBlank(message = "Card number is required")
    @Pattern(regexp = "^[0-9]{13,19}$", message = "Invalid card number")
    String cardNumber,

    @NotBlank(message = "CVV is required")
    @Pattern(regexp = "^[0-9]{3,4}$", message = "Invalid CVV")
    String cvv,

    @Email(message = "Invalid email")
    String email
) {}

// Custom validator
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SafeStringValidator.class)
public @interface SafeString {
    String message() default "String contains unsafe characters";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class SafeStringValidator implements ConstraintValidator<SafeString, String> {

    private static final Pattern UNSAFE_PATTERN =
        Pattern.compile("[<>\"'%;()&+]");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;
        return !UNSAFE_PATTERN.matcher(value).find();
    }
}
```

### XSS Prevention

```java
// ❌ BAD: Directly outputting user input
@GetMapping("/search")
public String search(@RequestParam String query, Model model) {
    model.addAttribute("query", query); // XSS vulnerability
    return "search";
}

// search.html
// <div th:text="${query}"></div> ← Still vulnerable!

// ✅ GOOD: Use Thymeleaf properly (auto-escapes)
// <div th:text="${query}"></div> ← Safe with th:text

// ✅ GOOD: Sanitize user input
@Service
public class HtmlSanitizer {

    private final PolicyFactory policy = Sanitizers.FORMATTING
        .and(Sanitizers.LINKS);

    public String sanitize(String untrustedHtml) {
        return policy.sanitize(untrustedHtml);
    }
}

// For JSON APIs, use proper Content-Type
@GetMapping("/api/search")
public ResponseEntity<SearchResponse> search(@RequestParam String query) {
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_JSON)
        .body(searchService.search(query));
}
```

---

## Authentication Security

### Password Policy

```java
@Component
public class PasswordPolicyValidator {

    private static final int MIN_LENGTH = 12;
    private static final Pattern UPPERCASE = Pattern.compile("[A-Z]");
    private static final Pattern LOWERCASE = Pattern.compile("[a-z]");
    private static final Pattern DIGIT = Pattern.compile("[0-9]");
    private static final Pattern SPECIAL = Pattern.compile("[!@#$%^&*]");

    public void validate(String password) {
        List<String> errors = new ArrayList<>();

        if (password.length() < MIN_LENGTH) {
            errors.add("Password must be at least " + MIN_LENGTH + " characters");
        }

        if (!UPPERCASE.matcher(password).find()) {
            errors.add("Password must contain uppercase letter");
        }

        if (!LOWERCASE.matcher(password).find()) {
            errors.add("Password must contain lowercase letter");
        }

        if (!DIGIT.matcher(password).find()) {
            errors.add("Password must contain digit");
        }

        if (!SPECIAL.matcher(password).find()) {
            errors.add("Password must contain special character");
        }

        // Check against common passwords
        if (isCommonPassword(password)) {
            errors.add("Password is too common");
        }

        if (!errors.isEmpty()) {
            throw new WeakPasswordException(errors);
        }
    }

    private boolean isCommonPassword(String password) {
        // Check against list of common passwords
        Set<String> commonPasswords = loadCommonPasswords();
        return commonPasswords.contains(password.toLowerCase());
    }
}
```

### Multi-Factor Authentication

```java
@Service
public class MfaService {

    private final TOTPService totpService;
    private final SmsService smsService;

    public MfaEnrollmentResponse enrollTotp(UserId userId) {
        // Generate secret
        String secret = totpService.generateSecret();

        // Save to user
        userRepository.updateMfaSecret(userId, secret);

        // Generate QR code
        String qrCodeUrl = totpService.generateQrCodeUrl(
            userId.getValue(),
            secret
        );

        return new MfaEnrollmentResponse(qrCodeUrl, secret);
    }

    public boolean verifyTotp(UserId userId, String code) {
        User user = userRepository.findById(userId)
            .orElseThrow();

        String secret = user.getMfaSecret();

        return totpService.verify(secret, code);
    }

    public void sendSmsMfa(UserId userId, String phoneNumber) {
        // Generate 6-digit code
        String code = generateSecureCode();

        // Store with TTL
        mfaCodeRepository.save(userId, code, Duration.ofMinutes(5));

        // Send SMS
        smsService.send(phoneNumber, "Your code is: " + code);
    }

    private String generateSecureCode() {
        SecureRandom random = new SecureRandom();
        return String.format("%06d", random.nextInt(1000000));
    }
}
```

### Rate Limiting Login Attempts

```java
@Service
public class LoginAttemptService {

    private final LoadingCache<String, Integer> attemptsCache;
    private final int MAX_ATTEMPTS = 5;
    private final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);

    public LoginAttemptService() {
        attemptsCache = CacheBuilder.newBuilder()
            .expireAfterWrite(LOCKOUT_DURATION)
            .build(new CacheLoader<>() {
                @Override
                public Integer load(String key) {
                    return 0;
                }
            });
    }

    public void loginSucceeded(String username) {
        attemptsCache.invalidate(username);
    }

    public void loginFailed(String username) {
        int attempts = attemptsCache.getUnchecked(username);
        attempts++;
        attemptsCache.put(username, attempts);

        if (attempts >= MAX_ATTEMPTS) {
            // Log security event
            log.warn("Account locked due to failed login attempts: {}", username);

            // Optional: Send notification
            securityEventPublisher.publish(
                new AccountLockedEvent(username, attempts)
            );
        }
    }

    public boolean isBlocked(String username) {
        return attemptsCache.getUnchecked(username) >= MAX_ATTEMPTS;
    }
}

// Login controller
@RestController
public class AuthController {

    @PostMapping("/api/auth/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {

        // Check if account is locked
        if (loginAttemptService.isBlocked(request.getUsername())) {
            throw new AccountLockedException(
                "Too many failed attempts. Try again later."
            );
        }

        try {
            LoginResponse response = authService.login(request);
            loginAttemptService.loginSucceeded(request.getUsername());
            return ResponseEntity.ok(response);

        } catch (BadCredentialsException e) {
            loginAttemptService.loginFailed(request.getUsername());
            throw e;
        }
    }
}
```

---

## Authorization Patterns

### Role-Based Access Control (RBAC)

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());

        return http.build();
    }
}

// Method-level security
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Order> getOrders(String userId) {
        return orderRepository.findByUserId(userId);
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(OrderId id) {
        orderRepository.deleteById(id);
    }

    @PostAuthorize("returnObject.userId == authentication.principal.id or hasRole('ADMIN')")
    public Order getOrder(OrderId id) {
        return orderRepository.findById(id)
            .orElseThrow();
    }
}
```

### Attribute-Based Access Control (ABAC)

```java
// Custom permission evaluator
@Component("orderSecurity")
public class OrderSecurityEvaluator {

    public boolean canAccess(Authentication authentication, OrderId orderId) {
        User user = (User) authentication.getPrincipal();

        Order order = orderRepository.findById(orderId)
            .orElseThrow();

        // Check ownership
        if (order.getUserId().equals(user.getId())) {
            return true;
        }

        // Check if user is admin
        if (user.hasRole("ADMIN")) {
            return true;
        }

        // Check if user is customer support AND order is not sensitive
        if (user.hasRole("SUPPORT") && !order.isSensitive()) {
            return true;
        }

        return false;
    }
}

// Usage
@GetMapping("/api/orders/{orderId}")
@PreAuthorize("@orderSecurity.canAccess(authentication, #orderId)")
public Order getOrder(@PathVariable OrderId orderId) {
    return orderService.getOrder(orderId);
}
```

---

## Cryptography

### Encryption at Rest

```java
@Service
public class EncryptionService {

    private final SecretKey key;

    public EncryptionService(@Value("${encryption.key}") String base64Key) {
        byte[] decodedKey = Base64.getDecoder().decode(base64Key);
        this.key = new SecretKeySpec(decodedKey, "AES");
    }

    public String encrypt(String plaintext) {
        try {
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");

            // Generate random IV
            byte[] iv = new byte[12];
            SecureRandom random = new SecureRandom();
            random.nextBytes(iv);
            GCMParameterSpec spec = new GCMParameterSpec(128, iv);

            cipher.init(Cipher.ENCRYPT_MODE, key, spec);
            byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));

            // Combine IV + ciphertext
            byte[] combined = new byte[iv.length + ciphertext.length];
            System.arraycopy(iv, 0, combined, 0, iv.length);
            System.arraycopy(ciphertext, 0, combined, iv.length, ciphertext.length);

            return Base64.getEncoder().encodeToString(combined);

        } catch (Exception e) {
            throw new EncryptionException("Encryption failed", e);
        }
    }

    public String decrypt(String encrypted) {
        try {
            byte[] combined = Base64.getDecoder().decode(encrypted);

            // Extract IV
            byte[] iv = new byte[12];
            System.arraycopy(combined, 0, iv, 0, iv.length);

            // Extract ciphertext
            byte[] ciphertext = new byte[combined.length - iv.length];
            System.arraycopy(combined, iv.length, ciphertext, 0, ciphertext.length);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            GCMParameterSpec spec = new GCMParameterSpec(128, iv);
            cipher.init(Cipher.DECRYPT_MODE, key, spec);

            byte[] plaintext = cipher.doFinal(ciphertext);
            return new String(plaintext, StandardCharsets.UTF_8);

        } catch (Exception e) {
            throw new EncryptionException("Decryption failed", e);
        }
    }
}

// JPA Converter for automatic encryption
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    @Autowired
    private EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        return encryptionService.encrypt(attribute);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        return encryptionService.decrypt(dbData);
    }
}

// Usage
@Entity
public class User {
    @Convert(converter = EncryptedStringConverter.class)
    private String socialSecurityNumber;
}
```

---

## Dependency Security

### Dependency Scanning

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>8.4.0</version>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </configuration>
</plugin>
```

```bash
# Run dependency check
mvn dependency-check:check

# Generate report
mvn dependency-check:aggregate

# Use Snyk
snyk test
snyk monitor

# GitHub Dependabot (automatic)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

---

## Production Hardening

### Security Headers

```java
@Configuration
public class SecurityHeadersConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'")
                )
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
                .frameOptions(frame -> frame.deny())
                .xssProtection(xss -> xss.block(true))
                .contentTypeOptions()
                .referrerPolicy(referrer -> referrer
                    .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                )
                .permissionsPolicy(permissions -> permissions
                    .policy("geolocation=(), microphone=(), camera=()")
                )
            );

        return http.build();
    }
}
```

### Logging Security Events

```java
@Component
public class SecurityAuditLogger {

    private static final Logger auditLog = LoggerFactory.getLogger("SECURITY_AUDIT");

    public void logAuthenticationSuccess(String username, String ipAddress) {
        auditLog.info("Authentication successful - user: {}, ip: {}",
            username, ipAddress);
    }

    public void logAuthenticationFailure(String username, String ipAddress, String reason) {
        auditLog.warn("Authentication failed - user: {}, ip: {}, reason: {}",
            username, ipAddress, reason);
    }

    public void logAuthorizationFailure(String username, String resource, String action) {
        auditLog.warn("Authorization denied - user: {}, resource: {}, action: {}",
            username, resource, action);
    }

    public void logSensitiveDataAccess(String username, String dataType, String recordId) {
        auditLog.info("Sensitive data accessed - user: {}, type: {}, id: {}",
            username, dataType, recordId);
    }
}
```

---

## Best Practices

### ✅ DO

- Validate all inputs
- Use parameterized queries
- Hash passwords with BCrypt
- Implement rate limiting
- Use HTTPS everywhere
- Keep dependencies updated
- Log security events
- Use security headers
- Implement least privilege
- Encrypt sensitive data

### ❌ DON'T

- Store passwords in plaintext
- Trust user input
- Roll your own crypto
- Expose stack traces
- Use weak encryption (MD5, SHA1)
- Hardcode secrets
- Ignore security updates
- Skip input validation
- Log sensitive data
- Use default credentials

---

*This guide provides production-ready security patterns for Java applications. Security is not optional - build it in from day one and maintain it continuously.*
