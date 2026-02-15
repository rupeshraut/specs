# Effective Modern Java Unit Testing

A comprehensive guide to mastering JUnit 5, Mockito, AssertJ, Testcontainers, and modern testing practices.

---

## Table of Contents

1. [Testing Fundamentals](#testing-fundamentals)
2. [JUnit 5 Basics](#junit-5-basics)
3. [Advanced JUnit 5 Features](#advanced-junit-5-features)
4. [Mockito Fundamentals](#mockito-fundamentals)
5. [Advanced Mocking Patterns](#advanced-mocking-patterns)
6. [AssertJ - Fluent Assertions](#assertj-fluent-assertions)
7. [Testcontainers - Integration Testing](#testcontainers-integration-testing)
8. [Spring Boot Testing](#spring-boot-testing)
9. [Testing Best Practices](#testing-best-practices)
10. [Test Doubles Patterns](#test-doubles-patterns)
11. [Property-Based Testing](#property-based-testing)
12. [Performance Testing](#performance-testing)
13. [Common Pitfalls](#common-pitfalls)
14. [Testing Checklist](#testing-checklist)

---

## Testing Fundamentals

### Test Pyramid

```
                    ┌─────────────┐
                    │     E2E     │  ← Few, slow, expensive
                    │   Tests     │
                    └─────────────┘
                  ┌─────────────────┐
                  │   Integration   │  ← Some, medium speed
                  │     Tests       │
                  └─────────────────┘
              ┌─────────────────────────┐
              │     Unit Tests          │  ← Many, fast, cheap
              │                         │
              └─────────────────────────┘

Unit Tests (70%):
- Test single unit in isolation
- Fast execution (< 100ms each)
- No external dependencies
- Use mocks/stubs

Integration Tests (20%):
- Test integration between components
- Real database, message queue, etc.
- Slower (seconds)
- Use Testcontainers

E2E Tests (10%):
- Test complete user flows
- Full application stack
- Slowest (minutes)
- Use Selenium, REST Assured
```

### Dependencies

```xml
<!-- Maven Dependencies -->
<properties>
    <junit.version>5.10.1</junit.version>
    <mockito.version>5.8.0</mockito.version>
    <assertj.version>3.24.2</assertj.version>
    <testcontainers.version>1.19.3</testcontainers.version>
</properties>

<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>${assertj.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## JUnit 5 Basics

### 1. **Basic Test Structure**

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    
    private Calculator calculator;
    
    // Runs once before all tests
    @BeforeAll
    static void setUpClass() {
        System.out.println("Setting up test class");
    }
    
    // Runs before each test
    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }
    
    // Runs after each test
    @AfterEach
    void tearDown() {
        calculator = null;
    }
    
    // Runs once after all tests
    @AfterAll
    static void tearDownClass() {
        System.out.println("Tearing down test class");
    }
    
    @Test
    void shouldAddTwoNumbers() {
        // Arrange
        int a = 5;
        int b = 3;
        
        // Act
        int result = calculator.add(a, b);
        
        // Assert
        assertEquals(8, result);
    }
    
    @Test
    @DisplayName("Subtraction: 5 - 3 = 2")
    void shouldSubtractTwoNumbers() {
        assertEquals(2, calculator.subtract(5, 3));
    }
    
    @Test
    void shouldThrowExceptionOnDivideByZero() {
        assertThrows(ArithmeticException.class, () -> {
            calculator.divide(10, 0);
        });
    }
    
    @Test
    @Disabled("Not implemented yet")
    void shouldCalculateComplexExpression() {
        // TODO: Implement
    }
}
```

### 2. **Assertions**

```java
class AssertionsExamples {
    
    @Test
    void basicAssertions() {
        // Equality
        assertEquals(4, 2 + 2);
        assertNotEquals(5, 2 + 2);
        
        // Boolean
        assertTrue(5 > 3);
        assertFalse(5 < 3);
        
        // Null checks
        assertNull(null);
        assertNotNull("not null");
        
        // Same object reference
        String str1 = "test";
        String str2 = str1;
        assertSame(str1, str2);
        
        // Array/Collection
        int[] expected = {1, 2, 3};
        int[] actual = {1, 2, 3};
        assertArrayEquals(expected, actual);
        
        // Timeout
        assertTimeout(Duration.ofSeconds(2), () -> {
            Thread.sleep(1000);
        });
        
        // Exception
        Exception exception = assertThrows(IllegalArgumentException.class, () -> {
            throw new IllegalArgumentException("Invalid argument");
        });
        assertEquals("Invalid argument", exception.getMessage());
    }
    
    @Test
    void groupedAssertions() {
        User user = new User("John", "Doe", 30);
        
        // All assertions execute even if some fail
        assertAll("User properties",
            () -> assertEquals("John", user.getFirstName()),
            () -> assertEquals("Doe", user.getLastName()),
            () -> assertEquals(30, user.getAge())
        );
    }
    
    @Test
    void customMessages() {
        int result = calculator.add(2, 2);
        
        // Static message
        assertEquals(4, result, "Addition should work");
        
        // Lazy message (only computed on failure)
        assertEquals(4, result, () -> 
            "Expected 4 but got " + result + " from complex calculation"
        );
    }
}
```

### 3. **Parameterized Tests**

```java
class ParameterizedTestExamples {
    
    // Simple value source
    @ParameterizedTest
    @ValueSource(strings = {"racecar", "radar", "level"})
    void shouldBePalindrome(String word) {
        assertTrue(isPalindrome(word));
    }
    
    // Multiple parameters
    @ParameterizedTest
    @CsvSource({
        "1, 1, 2",
        "2, 3, 5",
        "10, 20, 30"
    })
    void shouldAddNumbers(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
    
    // From CSV file
    @ParameterizedTest
    @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
    void shouldProcessCsvData(String name, int age, String email) {
        User user = new User(name, age, email);
        assertTrue(user.isValid());
    }
    
    // Method source
    @ParameterizedTest
    @MethodSource("provideUsersForTesting")
    void shouldValidateUser(User user) {
        assertTrue(validator.validate(user));
    }
    
    static Stream<User> provideUsersForTesting() {
        return Stream.of(
            new User("John", "Doe", 30),
            new User("Jane", "Smith", 25),
            new User("Bob", "Johnson", 40)
        );
    }
    
    // Enum source
    @ParameterizedTest
    @EnumSource(Month.class)
    void shouldHaveDaysInMonth(Month month) {
        assertTrue(month.length(false) >= 28);
        assertTrue(month.length(false) <= 31);
    }
    
    // Arguments source for complex objects
    @ParameterizedTest
    @MethodSource("provideInvalidEmails")
    void shouldRejectInvalidEmails(String email, String reason) {
        assertFalse(validator.isValidEmail(email), reason);
    }
    
    static Stream<Arguments> provideInvalidEmails() {
        return Stream.of(
            Arguments.of("invalid", "Missing @"),
            Arguments.of("@example.com", "Missing local part"),
            Arguments.of("user@", "Missing domain"),
            Arguments.of("user @example.com", "Contains space")
        );
    }
}
```

---

## Advanced JUnit 5 Features

### 1. **Nested Tests**

```java
@DisplayName("Order Service Tests")
class OrderServiceTest {
    
    private OrderService orderService;
    private Order order;
    
    @BeforeEach
    void setUp() {
        orderService = new OrderService();
    }
    
    @Nested
    @DisplayName("When order is new")
    class WhenNew {
        
        @BeforeEach
        void createNewOrder() {
            order = new Order();
        }
        
        @Test
        @DisplayName("Should be in PENDING status")
        void shouldBeInPendingStatus() {
            assertEquals(OrderStatus.PENDING, order.getStatus());
        }
        
        @Test
        @DisplayName("Should allow adding items")
        void shouldAllowAddingItems() {
            order.addItem(new OrderItem("Product", 1));
            assertEquals(1, order.getItems().size());
        }
        
        @Nested
        @DisplayName("After adding items")
        class AfterAddingItems {
            
            @BeforeEach
            void addItems() {
                order.addItem(new OrderItem("Product1", 2));
                order.addItem(new OrderItem("Product2", 1));
            }
            
            @Test
            @DisplayName("Should calculate total correctly")
            void shouldCalculateTotal() {
                BigDecimal total = orderService.calculateTotal(order);
                assertTrue(total.compareTo(BigDecimal.ZERO) > 0);
            }
            
            @Test
            @DisplayName("Should allow submission")
            void shouldAllowSubmission() {
                orderService.submit(order);
                assertEquals(OrderStatus.SUBMITTED, order.getStatus());
            }
        }
    }
    
    @Nested
    @DisplayName("When order is submitted")
    class WhenSubmitted {
        
        @BeforeEach
        void createSubmittedOrder() {
            order = new Order();
            order.addItem(new OrderItem("Product", 1));
            orderService.submit(order);
        }
        
        @Test
        @DisplayName("Should not allow adding items")
        void shouldNotAllowAddingItems() {
            assertThrows(IllegalStateException.class, () -> {
                order.addItem(new OrderItem("Another", 1));
            });
        }
        
        @Test
        @DisplayName("Should allow cancellation")
        void shouldAllowCancellation() {
            orderService.cancel(order);
            assertEquals(OrderStatus.CANCELLED, order.getStatus());
        }
    }
}
```

### 2. **Conditional Test Execution**

```java
class ConditionalTestExamples {
    
    // Operating System
    @Test
    @EnabledOnOs(OS.LINUX)
    void onlyOnLinux() {
        // Runs only on Linux
    }
    
    @Test
    @DisabledOnOs({OS.WINDOWS, OS.MAC})
    void notOnWindowsOrMac() {
        // Runs on all OS except Windows and Mac
    }
    
    // Java version
    @Test
    @EnabledOnJre(JRE.JAVA_17)
    void onlyOnJava17() {
        // Runs only on Java 17
    }
    
    @Test
    @EnabledForJreRange(min = JRE.JAVA_11, max = JRE.JAVA_17)
    void onJava11To17() {
        // Runs on Java 11 through 17
    }
    
    // Environment variables
    @Test
    @EnabledIfEnvironmentVariable(named = "ENV", matches = "production")
    void onlyInProduction() {
        // Runs only when ENV=production
    }
    
    // System properties
    @Test
    @EnabledIfSystemProperty(named = "os.arch", matches = ".*64.*")
    void only64Bit() {
        // Runs only on 64-bit systems
    }
    
    // Custom condition
    @Test
    @EnabledIf("customCondition")
    void enabledOnCustomCondition() {
        // Runs only if customCondition() returns true
    }
    
    boolean customCondition() {
        return LocalDate.now().getDayOfWeek() != DayOfWeek.SATURDAY;
    }
}
```

### 3. **Dynamic Tests**

```java
class DynamicTestExamples {
    
    @TestFactory
    Stream<DynamicTest> dynamicTestsFromStream() {
        return Stream.of("racecar", "radar", "level")
            .map(word -> DynamicTest.dynamicTest(
                "Testing palindrome: " + word,
                () -> assertTrue(isPalindrome(word))
            ));
    }
    
    @TestFactory
    Collection<DynamicTest> dynamicTestsFromCollection() {
        List<String> inputs = Arrays.asList("apple", "banana", "cherry");
        
        return inputs.stream()
            .map(input -> DynamicTest.dynamicTest(
                "Processing: " + input,
                () -> {
                    String processed = process(input);
                    assertNotNull(processed);
                    assertTrue(processed.length() > 0);
                }
            ))
            .collect(Collectors.toList());
    }
    
    @TestFactory
    Stream<DynamicNode> dynamicTestsWithContainers() {
        return Stream.of("positive", "negative", "zero")
            .map(category -> DynamicContainer.dynamicContainer(
                "Category: " + category,
                Stream.of(
                    DynamicTest.dynamicTest("Test 1", () -> assertTrue(true)),
                    DynamicTest.dynamicTest("Test 2", () -> assertTrue(true))
                )
            ));
    }
}
```

### 4. **Test Instance Lifecycle**

```java
// Default: New instance per test method
@TestInstance(TestInstance.Lifecycle.PER_METHOD)
class PerMethodLifecycleTest {
    
    private int counter = 0;
    
    @Test
    void test1() {
        counter++;
        assertEquals(1, counter);  // Always 1
    }
    
    @Test
    void test2() {
        counter++;
        assertEquals(1, counter);  // Always 1
    }
}

// One instance for all test methods
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class PerClassLifecycleTest {
    
    private int counter = 0;
    
    @BeforeAll  // Can be non-static with PER_CLASS
    void setUp() {
        // Initialize expensive resources once
    }
    
    @Test
    void test1() {
        counter++;
        assertEquals(1, counter);
    }
    
    @Test
    void test2() {
        counter++;
        assertEquals(2, counter);  // Shared state
    }
    
    @AfterAll  // Can be non-static with PER_CLASS
    void tearDown() {
        // Cleanup
    }
}
```

---

## Mockito Fundamentals

### 1. **Basic Mocking**

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    // Alternative: Manual setup
    @BeforeEach
    void setUp() {
        // userRepository = mock(UserRepository.class);
        // emailService = mock(EmailService.class);
        // userService = new UserService(userRepository, emailService);
    }
    
    @Test
    void shouldCreateUser() {
        // Arrange
        User user = new User("John", "john@example.com");
        when(userRepository.save(any(User.class))).thenReturn(user);
        
        // Act
        User created = userService.createUser(user);
        
        // Assert
        assertEquals("John", created.getName());
        verify(userRepository).save(user);
        verify(emailService).sendWelcomeEmail(user.getEmail());
    }
    
    @Test
    void shouldFindUserById() {
        // Arrange
        User user = new User("John", "john@example.com");
        when(userRepository.findById("123")).thenReturn(Optional.of(user));
        
        // Act
        Optional<User> found = userService.findById("123");
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("John", found.get().getName());
        verify(userRepository, times(1)).findById("123");
    }
    
    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        // Arrange
        when(userRepository.findById(anyString())).thenReturn(Optional.empty());
        
        // Act & Assert
        assertThrows(UserNotFoundException.class, () -> {
            userService.getUserOrThrow("999");
        });
        
        verify(userRepository).findById("999");
        verify(emailService, never()).sendWelcomeEmail(anyString());
    }
}
```

### 2. **Stubbing Variations**

```java
class StubbingExamples {
    
    @Mock
    private DataService dataService;
    
    @Test
    void returnValues() {
        // Simple return
        when(dataService.getData()).thenReturn("data");
        
        // Multiple calls
        when(dataService.getData())
            .thenReturn("first")
            .thenReturn("second")
            .thenReturn("third");
        
        assertEquals("first", dataService.getData());
        assertEquals("second", dataService.getData());
        assertEquals("third", dataService.getData());
    }
    
    @Test
    void throwExceptions() {
        when(dataService.getData())
            .thenThrow(new RuntimeException("Error"));
        
        assertThrows(RuntimeException.class, () -> dataService.getData());
    }
    
    @Test
    void customAnswer() {
        when(dataService.process(anyString())).thenAnswer(invocation -> {
            String arg = invocation.getArgument(0);
            return "Processed: " + arg;
        });
        
        assertEquals("Processed: test", dataService.process("test"));
    }
    
    @Test
    void doReturnStyle() {
        // Use doReturn when stubbing void methods or spy
        doReturn("data").when(dataService).getData();
        
        // void methods
        doNothing().when(dataService).clear();
        doThrow(new RuntimeException()).when(dataService).delete();
    }
    
    @Test
    void argumentMatchers() {
        // Any
        when(dataService.findById(anyInt())).thenReturn("found");
        
        // Specific value
        when(dataService.findById(eq(42))).thenReturn("specific");
        
        // Null
        when(dataService.findByName(isNull())).thenReturn("null-name");
        when(dataService.findByName(notNull())).thenReturn("has-name");
        
        // Custom matcher
        when(dataService.findByName(argThat(name -> name.length() > 5)))
            .thenReturn("long-name");
    }
}
```

### 3. **Verification**

```java
class VerificationExamples {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Test
    void verifyMethodCalls() {
        Order order = new Order();
        orderRepository.save(order);
        
        // Basic verification
        verify(orderRepository).save(order);
        
        // Never called
        verify(orderRepository, never()).delete(any());
        
        // Number of times
        verify(orderRepository, times(1)).save(order);
        verify(orderRepository, atLeastOnce()).save(order);
        verify(orderRepository, atLeast(1)).save(order);
        verify(orderRepository, atMost(5)).save(order);
        
        // Exact times
        orderRepository.save(order);
        verify(orderRepository, times(2)).save(order);
    }
    
    @Test
    void verifyOrder() {
        InOrder inOrder = inOrder(orderRepository);
        
        orderRepository.save(new Order());
        orderRepository.findAll();
        orderRepository.delete(new Order());
        
        inOrder.verify(orderRepository).save(any());
        inOrder.verify(orderRepository).findAll();
        inOrder.verify(orderRepository).delete(any());
    }
    
    @Test
    void verifyNoMoreInteractions() {
        orderRepository.findAll();
        
        verify(orderRepository).findAll();
        verifyNoMoreInteractions(orderRepository);
    }
    
    @Test
    void argumentCaptor() {
        ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
        
        Order order = new Order();
        order.setTotal(BigDecimal.TEN);
        orderRepository.save(order);
        
        verify(orderRepository).save(captor.capture());
        
        Order captured = captor.getValue();
        assertEquals(BigDecimal.TEN, captured.getTotal());
    }
}
```

---

## Advanced Mocking Patterns

### 1. **Spy - Partial Mocking**

```java
class SpyExamples {
    
    @Test
    void shouldUseRealMethodsByDefault() {
        List<String> list = new ArrayList<>();
        List<String> spy = spy(list);
        
        // Real methods called by default
        spy.add("one");
        spy.add("two");
        
        assertEquals(2, spy.size());  // Real method
        
        // Can stub specific methods
        when(spy.size()).thenReturn(100);
        assertEquals(100, spy.size());  // Stubbed
        
        // Original list not affected
        assertEquals(2, list.size());
    }
    
    @Spy
    private UserService userService;  // Real instance with @Spy
    
    @Test
    void shouldSpyOnRealObject() {
        // Real methods work
        User user = new User("John");
        userService.saveUser(user);
        
        // Can verify calls
        verify(userService).saveUser(user);
        
        // Can stub specific methods
        doReturn(true).when(userService).isEmailUnique("test@example.com");
        assertTrue(userService.isEmailUnique("test@example.com"));
    }
}
```

### 2. **Mocking Static Methods (Mockito 3.4+)**

```java
class StaticMockingExamples {
    
    @Test
    void shouldMockStaticMethod() {
        try (MockedStatic<Utils> utilities = mockStatic(Utils.class)) {
            // Stub static method
            utilities.when(() -> Utils.format("test"))
                .thenReturn("MOCKED");
            
            assertEquals("MOCKED", Utils.format("test"));
            
            // Verify static method call
            utilities.verify(() -> Utils.format("test"));
        }
        // Mock is automatically closed, real method restored
    }
    
    @Test
    void shouldMockStaticMethodConditionally() {
        try (MockedStatic<LocalDate> dateMock = mockStatic(LocalDate.class)) {
            LocalDate fixed = LocalDate.of(2024, 1, 15);
            
            dateMock.when(LocalDate::now).thenReturn(fixed);
            
            assertEquals(fixed, LocalDate.now());
        }
    }
}
```

### 3. **Mocking Constructors (Mockito 3.5+)**

```java
class ConstructorMockingExamples {
    
    @Test
    void shouldMockConstructor() {
        try (MockedConstruction<Database> mocked = mockConstruction(Database.class,
                (mock, context) -> {
                    when(mock.connect()).thenReturn(true);
                })) {
            
            Database db = new Database("localhost");
            assertTrue(db.connect());
            
            assertEquals(1, mocked.constructed().size());
        }
    }
}
```

### 4. **BDD Style with Mockito**

```java
import static org.mockito.BDDMockito.*;

class BDDStyleTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldProcessOrder() {
        // Given
        Order order = new Order();
        given(orderRepository.save(any(Order.class))).willReturn(order);
        
        // When
        Order result = orderService.processOrder(order);
        
        // Then
        then(orderRepository).should().save(order);
        then(orderRepository).should(never()).delete(any());
    }
}
```

---

## AssertJ - Fluent Assertions

### 1. **Basic Assertions**

```java
import static org.assertj.core.api.Assertions.*;

class AssertJBasics {
    
    @Test
    void basicAssertions() {
        // Strings
        assertThat("Hello World")
            .isNotNull()
            .startsWith("Hello")
            .endsWith("World")
            .contains("lo Wo")
            .hasSize(11);
        
        // Numbers
        assertThat(42)
            .isEqualTo(42)
            .isNotZero()
            .isPositive()
            .isGreaterThan(40)
            .isLessThan(50)
            .isBetween(40, 50);
        
        // Booleans
        assertThat(true)
            .isTrue()
            .isNotEqualTo(false);
        
        // Objects
        User user = new User("John", 30);
        assertThat(user)
            .isNotNull()
            .extracting("name", "age")
            .containsExactly("John", 30);
    }
    
    @Test
    void collectionAssertions() {
        List<String> list = Arrays.asList("one", "two", "three");
        
        assertThat(list)
            .isNotEmpty()
            .hasSize(3)
            .contains("two")
            .containsExactly("one", "two", "three")
            .containsExactlyInAnyOrder("three", "one", "two")
            .doesNotContain("four")
            .startsWith("one")
            .endsWith("three");
    }
    
    @Test
    void mapAssertions() {
        Map<String, Integer> map = Map.of("a", 1, "b", 2);
        
        assertThat(map)
            .isNotEmpty()
            .hasSize(2)
            .containsKey("a")
            .containsValue(1)
            .containsEntry("a", 1)
            .doesNotContainKey("c");
    }
    
    @Test
    void exceptionAssertions() {
        assertThatThrownBy(() -> {
            throw new IllegalArgumentException("Invalid");
        })
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Invalid")
            .hasMessageContaining("Invalid")
            .hasNoCause();
        
        // Alternative syntax
        assertThatExceptionOfType(IllegalArgumentException.class)
            .isThrownBy(() -> {
                throw new IllegalArgumentException("Invalid");
            })
            .withMessage("Invalid");
    }
}
```

### 2. **Advanced Assertions**

```java
class AssertJAdvanced {
    
    @Test
    void objectAssertions() {
        User user = new User("John", "Doe", 30);
        
        assertThat(user)
            .hasFieldOrPropertyWithValue("firstName", "John")
            .hasFieldOrPropertyWithValue("age", 30)
            .hasNoNullFieldsOrProperties();
        
        // Extracting fields
        assertThat(user)
            .extracting(User::getFirstName, User::getAge)
            .containsExactly("John", 30);
    }
    
    @Test
    void customAssertions() {
        User user = new User("John", "Doe", 30);
        
        assertThat(user)
            .satisfies(u -> {
                assertThat(u.getFirstName()).isEqualTo("John");
                assertThat(u.getAge()).isGreaterThan(18);
            });
    }
    
    @Test
    void filteringCollections() {
        List<User> users = Arrays.asList(
            new User("John", "Doe", 30),
            new User("Jane", "Smith", 25),
            new User("Bob", "Johnson", 40)
        );
        
        assertThat(users)
            .filteredOn(u -> u.getAge() > 25)
            .extracting(User::getFirstName)
            .containsExactly("John", "Bob");
        
        assertThat(users)
            .filteredOn("age", 30)
            .hasSize(1)
            .first()
            .extracting(User::getFirstName)
            .isEqualTo("John");
    }
    
    @Test
    void softAssertions() {
        // All assertions execute even if some fail
        SoftAssertions softly = new SoftAssertions();
        
        User user = new User("John", "Doe", 30);
        
        softly.assertThat(user.getFirstName()).isEqualTo("John");
        softly.assertThat(user.getLastName()).isEqualTo("Wrong");  // Fails
        softly.assertThat(user.getAge()).isEqualTo(25);  // Fails
        
        softly.assertAll();  // Reports all failures
    }
    
    @Test
    void assumptionBasedTests() {
        // Skip test if assumption fails
        assumeThat(System.getenv("ENV"))
            .isEqualTo("test");
        
        // Rest of test only runs if assumption passes
        User user = createUser();
        assertThat(user).isNotNull();
    }
}
```

---

## Testcontainers - Integration Testing

### 1. **Database Testing**

```java
@Testcontainers
class DatabaseIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    private DataSource dataSource;
    
    @BeforeEach
    void setUp() {
        dataSource = DataSourceBuilder.create()
            .url(postgres.getJdbcUrl())
            .username(postgres.getUsername())
            .password(postgres.getPassword())
            .build();
    }
    
    @Test
    void shouldSaveAndRetrieveUser() {
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        
        // Create table
        jdbc.execute("""
            CREATE TABLE users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100),
                email VARCHAR(100)
            )
            """);
        
        // Insert
        jdbc.update("INSERT INTO users (name, email) VALUES (?, ?)",
            "John Doe", "john@example.com");
        
        // Retrieve
        String name = jdbc.queryForObject(
            "SELECT name FROM users WHERE email = ?",
            String.class,
            "john@example.com"
        );
        
        assertThat(name).isEqualTo("John Doe");
    }
}
```

### 2. **Multiple Containers**

```java
@Testcontainers
class MultiContainerTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );
    
    @Test
    void shouldUseAllContainers() {
        // Use postgres
        String jdbcUrl = postgres.getJdbcUrl();
        
        // Use Redis
        String redisHost = redis.getHost();
        Integer redisPort = redis.getMappedPort(6379);
        
        // Use Kafka
        String bootstrapServers = kafka.getBootstrapServers();
        
        // Test integration
    }
}
```

### 3. **Docker Compose**

```java
@Testcontainers
class DockerComposeTest {
    
    @Container
    static ComposeContainer environment = new ComposeContainer(
            new File("src/test/resources/docker-compose.yml")
        )
        .withExposedService("postgres", 5432)
        .withExposedService("redis", 6379)
        .withLocalCompose(true);
    
    @Test
    void shouldUseComposeServices() {
        String postgresUrl = String.format(
            "jdbc:postgresql://%s:%d/testdb",
            environment.getServiceHost("postgres", 5432),
            environment.getServicePort("postgres", 5432)
        );
        
        // Test with services
    }
}
```

### 4. **Custom Containers**

```java
class CustomContainerExamples {
    
    static class MyAppContainer extends GenericContainer<MyAppContainer> {
        
        public MyAppContainer() {
            super(DockerImageName.parse("myapp:latest"));
            withExposedPorts(8080);
            withEnv("SPRING_PROFILES_ACTIVE", "test");
        }
        
        public String getBaseUrl() {
            return "http://" + getHost() + ":" + getMappedPort(8080);
        }
    }
    
    @Container
    static MyAppContainer app = new MyAppContainer();
    
    @Test
    void shouldConnectToApp() {
        RestTemplate restTemplate = new RestTemplate();
        String response = restTemplate.getForObject(
            app.getBaseUrl() + "/health",
            String.class
        );
        
        assertThat(response).contains("UP");
    }
}
```

---

## Spring Boot Testing

### 1. **Spring Boot Test Slices**

```java
// Full application context
@SpringBootTest
class FullContextTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    void shouldCreateUser() {
        User user = userService.createUser(new User("John"));
        assertThat(user.getId()).isNotNull();
    }
}

// Web layer only
@WebMvcTest(UserController.class)
class WebLayerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void shouldReturnUser() throws Exception {
        User user = new User("John");
        when(userService.findById("1")).thenReturn(Optional.of(user));
        
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}

// JPA layer only
@DataJpaTest
class JpaLayerTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    void shouldSaveAndFindUser() {
        User user = new User("John");
        entityManager.persist(user);
        entityManager.flush();
        
        Optional<User> found = userRepository.findByName("John");
        
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}

// JSON serialization
@JsonTest
class JsonTest {
    
    @Autowired
    private JacksonTester<User> json;
    
    @Test
    void shouldSerializeUser() throws Exception {
        User user = new User("John", "john@example.com");
        
        assertThat(json.write(user))
            .hasJsonPathStringValue("$.name", "John")
            .hasJsonPathStringValue("$.email", "john@example.com");
    }
    
    @Test
    void shouldDeserializeUser() throws Exception {
        String content = "{\"name\":\"John\",\"email\":\"john@example.com\"}";
        
        User user = json.parse(content).getObject();
        
        assertThat(user.getName()).isEqualTo("John");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }
}
```

### 2. **Test Configuration**

```java
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary
    public Clock fixedClock() {
        return Clock.fixed(
            Instant.parse("2024-01-15T10:00:00Z"),
            ZoneId.of("UTC")
        );
    }
    
    @Bean
    public RestTemplate testRestTemplate() {
        return new RestTemplate();
    }
}

@SpringBootTest
@Import(TestConfig.class)
class ServiceWithTestConfig {
    
    @Autowired
    private Clock clock;
    
    @Test
    void shouldUseFixedClock() {
        Instant now = Instant.now(clock);
        assertThat(now).isEqualTo(Instant.parse("2024-01-15T10:00:00Z"));
    }
}
```

### 3. **Integration Testing with Testcontainers**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class IntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldCreateAndRetrieveUser() {
        // Create
        User user = new User("John", "john@example.com");
        ResponseEntity<User> createResponse = restTemplate.postForEntity(
            "/api/users",
            user,
            User.class
        );
        
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String userId = createResponse.getBody().getId();
        
        // Retrieve
        ResponseEntity<User> getResponse = restTemplate.getForEntity(
            "/api/users/" + userId,
            User.class
        );
        
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getName()).isEqualTo("John");
    }
}
```

---

## Testing Best Practices

### 1. **AAA Pattern (Arrange-Act-Assert)**

```java
class AAAPatternExamples {
    
    @Test
    void shouldFollowAAAPattern() {
        // Arrange - Set up test data and dependencies
        User user = new User("John", "john@example.com");
        UserRepository repository = mock(UserRepository.class);
        when(repository.save(any())).thenReturn(user);
        UserService service = new UserService(repository);
        
        // Act - Execute the code under test
        User result = service.createUser(user);
        
        // Assert - Verify the outcome
        assertThat(result.getName()).isEqualTo("John");
        verify(repository).save(user);
    }
}
```

### 2. **Test Naming Conventions**

```java
class TestNamingExamples {
    
    // Method name pattern: should<ExpectedBehavior>When<StateUnderTest>
    
    @Test
    void shouldReturnUserWhenUserExists() {
        // Test implementation
    }
    
    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        // Test implementation
    }
    
    @Test
    void shouldCalculateTotalWhenOrderHasItems() {
        // Test implementation
    }
    
    // BDD style with @DisplayName
    @Test
    @DisplayName("Given valid user, When creating user, Then user is saved")
    void givenValidUser_whenCreatingUser_thenUserIsSaved() {
        // Test implementation
    }
}
```

### 3. **Test Data Builders**

```java
class TestDataBuilder {
    
    // Builder pattern for test data
    static class UserBuilder {
        private String name = "John Doe";
        private String email = "john@example.com";
        private int age = 30;
        private List<Address> addresses = new ArrayList<>();
        
        public static UserBuilder aUser() {
            return new UserBuilder();
        }
        
        public UserBuilder withName(String name) {
            this.name = name;
            return this;
        }
        
        public UserBuilder withEmail(String email) {
            this.email = email;
            return this;
        }
        
        public UserBuilder withAge(int age) {
            this.age = age;
            return this;
        }
        
        public UserBuilder withAddress(Address address) {
            this.addresses.add(address);
            return this;
        }
        
        public User build() {
            User user = new User(name, email, age);
            user.setAddresses(addresses);
            return user;
        }
    }
    
    @Test
    void shouldUseTestDataBuilder() {
        // Clean, readable test data creation
        User user = UserBuilder.aUser()
            .withName("Jane Smith")
            .withEmail("jane@example.com")
            .withAge(25)
            .withAddress(new Address("123 Main St"))
            .build();
        
        assertThat(user.getName()).isEqualTo("Jane Smith");
        assertThat(user.getAddresses()).hasSize(1);
    }
}
```

### 4. **Object Mother Pattern**

```java
class TestDataFactory {
    
    // Centralized test data creation
    public static class Users {
        
        public static User johnDoe() {
            return new User("John", "Doe", "john@example.com", 30);
        }
        
        public static User janeSmith() {
            return new User("Jane", "Smith", "jane@example.com", 25);
        }
        
        public static User adminUser() {
            User user = new User("Admin", "User", "admin@example.com", 40);
            user.setRole(Role.ADMIN);
            return user;
        }
        
        public static User withRandomEmail() {
            return new User(
                "Test",
                "User",
                "test" + UUID.randomUUID() + "@example.com",
                25
            );
        }
    }
    
    @Test
    void shouldUseObjectMother() {
        User user = Users.johnDoe();
        
        assertThat(user.getFirstName()).isEqualTo("John");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }
}
```

---

## Test Doubles Patterns

### 1. **Test Doubles Overview**

```java
/**
 * Test Double Types:
 * 
 * 1. Dummy - Passed but never used
 * 2. Stub - Provides predefined answers
 * 3. Spy - Records information about calls
 * 4. Mock - Verifies behavior
 * 5. Fake - Working implementation (simplified)
 */

class TestDoublesExamples {
    
    // Dummy - Never used
    @Test
    void dummyExample() {
        EmailService dummy = mock(EmailService.class);
        // Passed to constructor but never called
        UserService service = new UserService(repository, dummy);
    }
    
    // Stub - Returns predefined values
    @Test
    void stubExample() {
        UserRepository stub = mock(UserRepository.class);
        when(stub.findById(anyString())).thenReturn(Optional.of(new User()));
        
        // Just returns values, no verification
    }
    
    // Spy - Real object with selective stubbing
    @Test
    void spyExample() {
        UserService spy = spy(new UserService());
        doReturn(true).when(spy).isEmailUnique("test@example.com");
        
        // Can verify calls
        spy.validateEmail("test@example.com");
        verify(spy).isEmailUnique("test@example.com");
    }
    
    // Mock - Behavior verification
    @Test
    void mockExample() {
        UserRepository mock = mock(UserRepository.class);
        UserService service = new UserService(mock);
        
        service.deleteUser("123");
        
        // Verify behavior
        verify(mock).deleteById("123");
    }
    
    // Fake - Simplified working implementation
    static class FakeUserRepository implements UserRepository {
        private final Map<String, User> users = new HashMap<>();
        
        @Override
        public Optional<User> findById(String id) {
            return Optional.ofNullable(users.get(id));
        }
        
        @Override
        public User save(User user) {
            users.put(user.getId(), user);
            return user;
        }
    }
    
    @Test
    void fakeExample() {
        UserRepository fake = new FakeUserRepository();
        UserService service = new UserService(fake);
        
        User user = new User("123", "John");
        service.createUser(user);
        
        Optional<User> found = fake.findById("123");
        assertThat(found).isPresent();
    }
}
```

---

## Property-Based Testing

### 1. **JUnit QuickCheck**

```java
import net.jqwik.api.*;

class PropertyBasedTests {
    
    @Property
    void absoluteValueAlwaysPositive(@ForAll int number) {
        int result = Math.abs(number);
        assertThat(result).isGreaterThanOrEqualTo(0);
    }
    
    @Property
    void reverseTwiceIsIdentity(@ForAll List<Integer> list) {
        List<Integer> reversed = reverse(reverse(list));
        assertThat(reversed).isEqualTo(list);
    }
    
    @Property
    void additionIsCommutative(@ForAll int a, @ForAll int b) {
        assertThat(a + b).isEqualTo(b + a);
    }
    
    @Property
    void stringConcatenationLength(
            @ForAll String s1,
            @ForAll String s2) {
        
        String concatenated = s1 + s2;
        assertThat(concatenated.length())
            .isEqualTo(s1.length() + s2.length());
    }
    
    @Property
    void validEmailPattern(
            @ForAll @Email String email) {
        
        assertThat(email).contains("@");
        assertThat(email).matches("^[^@]+@[^@]+\\.[^@]+$");
    }
}
```

---

## Performance Testing

### 1. **JMH Benchmarks**

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class PerformanceBenchmark {
    
    private List<String> list;
    
    @Setup
    public void setUp() {
        list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            list.add("item" + i);
        }
    }
    
    @Benchmark
    public void testArrayListIteration() {
        for (String item : list) {
            // Process item
        }
    }
    
    @Benchmark
    public void testStreamIteration() {
        list.stream().forEach(item -> {
            // Process item
        });
    }
}
```

### 2. **Performance Assertions**

```java
class PerformanceTests {
    
    @Test
    void shouldExecuteWithinTimeLimit() {
        assertTimeout(Duration.ofMillis(100), () -> {
            // Code that should complete within 100ms
            expensiveOperation();
        });
    }
    
    @Test
    void shouldMeasureExecutionTime() {
        long startTime = System.nanoTime();
        
        performOperation();
        
        long duration = System.nanoTime() - startTime;
        
        assertThat(duration)
            .isLessThan(Duration.ofMillis(100).toNanos());
    }
}
```

---

## Common Pitfalls

### ❌ **Anti-Patterns to Avoid**

```java
class CommonPitfalls {
    
    // ❌ BAD: Testing implementation details
    @Test
    void badTest_testingPrivateMethod() {
        // Don't test private methods directly
        // Test public API instead
    }
    
    // ✅ GOOD: Test public behavior
    @Test
    void goodTest_testPublicBehavior() {
        UserService service = new UserService();
        User user = service.createUser(new User("John"));
        
        assertThat(user.isValid()).isTrue();
    }
    
    // ❌ BAD: Depending on test execution order
    private static int counter = 0;
    
    @Test
    void badTest1() {
        counter++;
        assertEquals(1, counter);  // Breaks if order changes
    }
    
    // ✅ GOOD: Each test is independent
    @Test
    void goodTest() {
        int localCounter = 0;
        localCounter++;
        assertEquals(1, localCounter);
    }
    
    // ❌ BAD: Catching exceptions instead of asserting
    @Test
    void badTest_catchingException() {
        try {
            service.deleteUser("invalid");
            fail("Should have thrown exception");
        } catch (Exception e) {
            // This works but is verbose
        }
    }
    
    // ✅ GOOD: Use assertThrows
    @Test
    void goodTest_assertThrows() {
        assertThrows(UserNotFoundException.class, () -> {
            service.deleteUser("invalid");
        });
    }
    
    // ❌ BAD: Ignoring test failures
    @Test
    @Disabled("Fails intermittently")
    void badTest_ignored() {
        // Fix the test instead of ignoring it!
    }
    
    // ❌ BAD: Testing too much in one test
    @Test
    void badTest_testingEverything() {
        User user = service.createUser(new User());
        service.updateUser(user);
        service.deleteUser(user.getId());
        List<User> users = service.findAll();
        // Too many concerns in one test
    }
    
    // ✅ GOOD: One concept per test
    @Test
    void shouldCreateUser() {
        User user = service.createUser(new User());
        assertThat(user.getId()).isNotNull();
    }
    
    @Test
    void shouldUpdateUser() {
        User user = existingUser();
        service.updateUser(user);
        // Verify update
    }
}
```

---

## Testing Checklist

### ✅ **Unit Test Checklist**

```
Test Coverage:
□ Test happy paths
□ Test edge cases
□ Test error conditions
□ Test boundary values
□ Test null handling

Test Quality:
□ Tests are fast (< 100ms)
□ Tests are independent
□ Tests are repeatable
□ Tests follow AAA pattern
□ Tests have meaningful names

Test Isolation:
□ No external dependencies
□ Mocks are verified
□ State is reset between tests
□ No test execution order dependency

Assertions:
□ Use AssertJ for fluent assertions
□ Assert specific values, not implementation
□ One logical assertion per test
□ Meaningful assertion messages

Code Quality:
□ Tests are readable
□ Test data is clear
□ No code duplication
□ Tests follow coding standards
```

### Best Practices Summary

#### ✅ DO

1. **Write tests first (TDD)** - Design emerges from tests
2. **Test behavior, not implementation** - Tests shouldn't break on refactoring
3. **Use descriptive test names** - `shouldReturnUserWhenValidId()`
4. **Follow AAA pattern** - Arrange, Act, Assert
5. **One assertion per test** - Or related assertions
6. **Use test data builders** - For complex objects
7. **Mock external dependencies** - Database, network, time
8. **Use AssertJ** - More readable assertions
9. **Test edge cases** - Null, empty, negative, boundary
10. **Keep tests fast** - Unit tests < 100ms

#### ❌ DON'T

1. **Don't test private methods** - Test through public API
2. **Don't depend on test order** - Each test independent
3. **Don't ignore failing tests** - Fix or delete them
4. **Don't over-mock** - Use real objects when simple
5. **Don't test framework code** - Spring, Hibernate, etc.
6. **Don't hardcode values** - Use constants or variables
7. **Don't write integration tests as unit tests** - Keep them separate
8. **Don't skip edge cases** - They find most bugs
9. **Don't make tests complex** - Simple, readable tests
10. **Don't test getters/setters** - Unless they have logic

---

## Conclusion

**Modern Java Testing Stack:**

1. **JUnit 5** - Test framework (parameterized, nested, dynamic tests)
2. **Mockito** - Mocking framework (mocks, spies, verification)
3. **AssertJ** - Fluent assertions (readable, powerful)
4. **Testcontainers** - Integration testing (real dependencies)
5. **Spring Boot Test** - Spring integration (test slices)

**Key Principles:**

- **Fast Feedback** - Unit tests run in milliseconds
- **Isolation** - No external dependencies
- **Repeatability** - Same result every time
- **Readability** - Tests are documentation
- **Maintainability** - Easy to update when code changes

**Testing Pyramid:**
- 70% Unit Tests (fast, isolated, many)
- 20% Integration Tests (medium speed, some)
- 10% E2E Tests (slow, expensive, few)

**Remember**: Good tests are an investment. They:
- Catch bugs early
- Enable refactoring
- Document behavior
- Improve design
- Build confidence

Master these tools and patterns, and you'll write tests that add value, not burden!
