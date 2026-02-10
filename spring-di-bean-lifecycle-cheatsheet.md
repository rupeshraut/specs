# 🌱 Spring DI & Bean Lifecycle Cheat Sheet

> **Purpose:** Production patterns for dependency injection, bean lifecycle, auto-configuration, and conditional wiring. Reference before designing any Spring configuration or custom starter.
> **Stack context:** Java 21+ / Spring Boot 3.x / Spring Framework 6.x

---

## 📋 The DI Decision Framework

| Question | Determines |
|----------|-----------|
| Does this bean have **state**? | Scope: singleton vs prototype |
| Is it **always needed** or only sometimes? | Conditional / lazy loading |
| Does it depend on **which profile** is active? | @Profile / @ConditionalOnProperty |
| Does it need **initialization** after construction? | @PostConstruct / InitializingBean |
| Does it need **cleanup** before shutdown? | @PreDestroy / DisposableBean |
| Should it be **replaceable** in tests? | Interface + @Primary / @Qualifier |

---

## 🔄 Pattern 1: Bean Lifecycle

### Complete Lifecycle Sequence

```
 1. BeanDefinition loaded from @Configuration / @Component scan
 2. BeanFactoryPostProcessor runs (modify definitions before instantiation)
 3. Constructor called (dependency injection happens here)
 4. @Autowired setter/field injection (avoid — use constructor)
 5. BeanPostProcessor.postProcessBeforeInitialization
 6. @PostConstruct method
 7. InitializingBean.afterPropertiesSet()
 8. @Bean(initMethod = "init")
 9. BeanPostProcessor.postProcessAfterInitialization (proxies created: AOP, @Transactional)
10. ─── Bean is READY ───
11. ApplicationReadyEvent fired (all beans initialized)
12. ─── Application RUNNING ───
13. ContextClosedEvent (shutdown initiated)
14. @PreDestroy method
15. DisposableBean.destroy()
16. @Bean(destroyMethod = "cleanup")
```

### Lifecycle Hooks — When to Use Each

```java
@Component
public class PaymentGatewayClient {

    private final PaymentProperties properties;
    private HttpClient httpClient;

    // ── Constructor: inject dependencies (PREFERRED for all DI) ──
    public PaymentGatewayClient(PaymentProperties properties) {
        this.properties = properties;
        // DON'T do I/O or heavy init here — use @PostConstruct
    }

    // ── @PostConstruct: one-time initialization after DI complete ──
    @PostConstruct
    void initialize() {
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(properties.gateway().connectTimeout())
            .build();
        log.info("Payment gateway client initialized: {}", properties.gateway().baseUrl());
    }

    // ── @PreDestroy: cleanup before shutdown ──
    @PreDestroy
    void shutdown() {
        log.info("Shutting down payment gateway client");
        // Close connections, flush buffers, etc.
    }
}
```

### Application Events — Ordered Hooks

```java
@Component
public class StartupInitializer {

    // Fires AFTER all beans created but BEFORE accepting traffic
    @EventListener(ApplicationStartedEvent.class)
    public void onStarted() {
        log.info("Application started — performing pre-traffic initialization");
        // Warm caches, verify connections, run health checks
    }

    // Fires AFTER application is fully ready (including actuator)
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        log.info("Application ready — accepting traffic");
        // Start Kafka consumers, enable scheduled tasks
    }

    // Fires when context is closing
    @EventListener(ContextClosedEvent.class)
    public void onShutdown() {
        log.info("Application shutting down");
        // Drain queues, flush outbox, close connections
    }
}
```

---

## 💉 Pattern 2: Injection Patterns

### Constructor Injection (Always Use This)

```java
// ✅ RECOMMENDED — immutable, testable, explicit dependencies
@Service
public class PaymentService {

    private final PaymentRepository repository;
    private final PaymentGatewayPort gateway;
    private final PaymentEventPublisher eventPublisher;

    // Single constructor — @Autowired is implicit in Spring Boot
    public PaymentService(
            PaymentRepository repository,
            PaymentGatewayPort gateway,
            PaymentEventPublisher eventPublisher) {
        this.repository = repository;
        this.gateway = gateway;
        this.eventPublisher = eventPublisher;
    }
}

// ❌ AVOID — field injection (untestable without reflection)
@Service
public class PaymentService {
    @Autowired private PaymentRepository repository;   // Can't mock in unit test!
    @Autowired private PaymentGatewayPort gateway;
}

// ❌ AVOID — setter injection (mutable, forgettable)
@Service
public class PaymentService {
    private PaymentRepository repository;

    @Autowired
    public void setRepository(PaymentRepository repository) {
        this.repository = repository;   // Can be called multiple times, nullable
    }
}
```

### Multiple Implementations — Qualifier & Primary

```java
// Interface with multiple implementations
public interface NotificationSender {
    void send(String recipient, String message);
}

@Component("emailSender")
public class EmailNotificationSender implements NotificationSender { ... }

@Component("smsSender")
public class SmsNotificationSender implements NotificationSender { ... }

@Component("pushSender")
@Primary   // Default when no qualifier specified
public class PushNotificationSender implements NotificationSender { ... }

// ── Option 1: @Qualifier for specific implementation ──
@Service
public class AlertService {
    public AlertService(@Qualifier("emailSender") NotificationSender sender) { ... }
}

// ── Option 2: Inject ALL implementations ──
@Service
public class NotificationRouter {
    private final List<NotificationSender> allSenders;  // Spring injects all 3

    public NotificationRouter(List<NotificationSender> allSenders) {
        this.allSenders = allSenders;
    }
}

// ── Option 3: Registry pattern (best for strategy selection) ──
@Service
public class NotificationRegistry {
    private final Map<String, NotificationSender> senders;

    public NotificationRegistry(Map<String, NotificationSender> senders) {
        this.senders = senders;  // Keys are bean names
    }

    public NotificationSender get(String channel) {
        return senders.get(channel + "Sender");
    }
}
```

### Optional Dependencies

```java
@Service
public class PaymentService {

    private final PaymentRepository repository;
    private final Optional<FraudCheckPort> fraudCheck;   // May not be configured
    private final MetricsRecorder metrics;

    public PaymentService(
            PaymentRepository repository,
            Optional<FraudCheckPort> fraudCheck,          // Optional DI
            @Nullable MetricsRecorder metrics) {          // Nullable alternative
        this.repository = repository;
        this.fraudCheck = fraudCheck;
        this.metrics = metrics != null ? metrics : MetricsRecorder.NOOP;
    }

    public PaymentResult process(Payment payment) {
        fraudCheck.ifPresent(fc -> fc.evaluate(payment));  // Only if bean exists
        // ...
    }
}
```

---

## ⚙️ Pattern 3: Conditional Beans

### @Conditional Variants

```java
@Configuration
public class InfrastructureConfig {

    // ── Only if property is set ──
    @Bean
    @ConditionalOnProperty(name = "payment.gateway.enabled", havingValue = "true")
    public PaymentGatewayPort paymentGateway(PaymentProperties props) {
        return new StripePaymentGateway(props.gateway().baseUrl());
    }

    // ── Only if class is on classpath ──
    @Bean
    @ConditionalOnClass(name = "io.confluent.kafka.serializers.KafkaAvroSerializer")
    public KafkaAvroSerializer avroSerializer() {
        return new KafkaAvroSerializer();
    }

    // ── Only if another bean exists ──
    @Bean
    @ConditionalOnBean(PaymentGatewayPort.class)
    public PaymentGatewayHealthIndicator gatewayHealth(PaymentGatewayPort gateway) {
        return new PaymentGatewayHealthIndicator(gateway);
    }

    // ── Only if bean is MISSING (provide default) ──
    @Bean
    @ConditionalOnMissingBean(PaymentEventPublisher.class)
    public PaymentEventPublisher defaultEventPublisher() {
        return event -> log.info("Event published (noop): {}", event);
    }

    // ── Profile-based ──
    @Bean
    @Profile("!prod")    // Every profile EXCEPT prod
    public PaymentGatewayPort sandboxGateway() {
        return new SandboxPaymentGateway();
    }

    @Bean
    @Profile("prod")
    public PaymentGatewayPort productionGateway(PaymentProperties props) {
        return new StripePaymentGateway(props.gateway().baseUrl());
    }

    // ── Custom condition ──
    @Bean
    @Conditional(MultiDatacenterCondition.class)
    public DatacenterAwareRouter dcRouter() {
        return new DatacenterAwareRouter();
    }
}

// Custom condition implementation
public class MultiDatacenterCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata metadata) {
        String dcConfig = ctx.getEnvironment().getProperty("datacenter.mode");
        return "multi".equals(dcConfig);
    }
}
```

---

## 🏭 Pattern 4: Custom Auto-Configuration

### Building a Reusable Starter

```java
// ── Auto-configuration class ──
@AutoConfiguration
@ConditionalOnClass(PaymentGatewayPort.class)
@EnableConfigurationProperties(PaymentGatewayProperties.class)
public class PaymentGatewayAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PaymentGatewayPort paymentGateway(PaymentGatewayProperties properties) {
        return new DefaultPaymentGateway(
            properties.baseUrl(),
            properties.connectTimeout(),
            properties.readTimeout()
        );
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean(PaymentGatewayPort.class)
    public PaymentGatewayHealthIndicator gatewayHealthIndicator(PaymentGatewayPort gateway) {
        return new PaymentGatewayHealthIndicator(gateway);
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean({PaymentGatewayPort.class, MeterRegistry.class})
    public PaymentGatewayMetrics gatewayMetrics(MeterRegistry registry) {
        return new PaymentGatewayMetrics(registry);
    }
}

// ── Type-safe properties ──
@ConfigurationProperties(prefix = "payment.gateway")
@Validated
public record PaymentGatewayProperties(
    @NotBlank String baseUrl,
    @DurationMin(millis = 100) Duration connectTimeout,
    @DurationMin(millis = 100) Duration readTimeout,
    int maxConnections
) {
    public PaymentGatewayProperties {
        if (connectTimeout == null) connectTimeout = Duration.ofSeconds(3);
        if (readTimeout == null) readTimeout = Duration.ofSeconds(10);
        if (maxConnections <= 0) maxConnections = 50;
    }
}

// ── Registration file ──
// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// com.example.starter.PaymentGatewayAutoConfiguration
```

---

## 🔄 Pattern 5: Bean Scopes

```
SINGLETON (default)
  One instance per ApplicationContext. Shared across all injections.
  Use for: Stateless services, repositories, configs, clients.
  ⚠️ Must be THREAD-SAFE (no mutable instance fields).

PROTOTYPE
  New instance on every injection / every getBean() call.
  Use for: Stateful objects (builders, request-scoped state).
  ⚠️ Spring does NOT manage lifecycle (no @PreDestroy).

REQUEST (web only)
  One instance per HTTP request.
  Use for: Request-scoped context, audit trails.
  ⚠️ Not available outside web request context.

SESSION (web only)
  One instance per HTTP session.
  Use for: User preferences, shopping cart.

APPLICATION
  One instance per ServletContext.
  Rarely used — singleton is almost always sufficient.
```

### Injecting Prototype into Singleton

```java
// ❌ WRONG: prototype bean injected once at singleton creation time
@Service
public class PaymentProcessor {
    private final PaymentContext context;  // Same instance forever!
    public PaymentProcessor(PaymentContext context) {
        this.context = context;
    }
}

// ✅ Option 1: ObjectProvider (lazy lookup)
@Service
public class PaymentProcessor {
    private final ObjectProvider<PaymentContext> contextProvider;

    public PaymentProcessor(ObjectProvider<PaymentContext> contextProvider) {
        this.contextProvider = contextProvider;
    }

    public void process(Payment payment) {
        var context = contextProvider.getObject();  // New instance each time
        context.initialize(payment);
        // ...
    }
}

// ✅ Option 2: @Lookup method
@Service
public abstract class PaymentProcessor {
    @Lookup
    protected abstract PaymentContext createContext();  // Spring overrides this

    public void process(Payment payment) {
        var context = createContext();  // New instance each time
    }
}
```

---

## 🧪 Pattern 6: Testing Configuration

```java
// ── Override beans in tests ──
@SpringBootTest
class PaymentServiceIntegrationTest {

    // Overrides the real PaymentGatewayPort bean for this test
    @TestConfiguration
    static class TestConfig {
        @Bean
        @Primary
        public PaymentGatewayPort testGateway() {
            return new StubPaymentGateway();
        }
    }
}

// ── Selective context loading ──
@SpringBootTest(classes = {
    PaymentService.class,
    PaymentRepository.class,
    TestMongoConfig.class     // Only load what's needed
})
class PaymentServiceSliceTest { ... }

// ── @MockBean for specific replacement ──
@SpringBootTest
class PaymentControllerTest {
    @MockBean private PaymentGatewayPort gateway;  // Replaces real bean with mock

    @Test
    void shouldProcessPayment() {
        when(gateway.charge(any())).thenReturn(ChargeResult.success("tx-1"));
        // ...
    }
}
```

---

## 🚫 DI Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Field injection** | Untestable, mutable, hides dependencies | Constructor injection |
| **Circular dependencies** | Design smell, fragile initialization | Redesign or use @Lazy on one side |
| **Too many constructor params** (>7) | Class does too much (SRP violation) | Split into smaller services |
| **@Autowired on concrete class** | Tight coupling to implementation | Inject interface |
| **Singleton with mutable state** | Thread safety issues | Make stateless or use prototype |
| **@PostConstruct with I/O that can fail** | Prevents application startup | Use ApplicationReadyEvent + retry |
| **BeanFactory.getBean() in business code** | Service locator anti-pattern | Use constructor injection |
| **@Value scattered everywhere** | Magic strings, hard to track | @ConfigurationProperties record |

---

## 💡 Golden Rules of Spring DI

```
1.  CONSTRUCTOR INJECTION always — immutable, testable, explicit.
2.  PROGRAM TO INTERFACES — inject PaymentGatewayPort, not StripeGateway.
3.  @ConfigurationProperties records > @Value — type-safe, validated, discoverable.
4.  @ConditionalOnMissingBean = extensibility — let users override your defaults.
5.  Singleton beans MUST be thread-safe — no mutable instance fields.
6.  @PostConstruct for init, @PreDestroy for cleanup, ApplicationReadyEvent for "start accepting traffic."
7.  SMALL beans > large beans — if constructor has 7+ params, the class does too much.
8.  @Primary for defaults, @Qualifier for specifics, List<> for all implementations.
9.  Test with @TestConfiguration + @Primary — override specific beans, not the whole context.
10. Auto-configuration = reusability — extract shared patterns into starters.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / Spring Framework 6.x*
