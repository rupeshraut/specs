# Effective Modern Java Functional Interfaces

A comprehensive guide to mastering `java.util.function` package, functional programming patterns, and writing clean, composable code.

---

## Table of Contents

1. [Functional Interfaces Overview](#functional-interfaces-overview)
2. [Function\<T, R\>](#functiont-r)
3. [Predicate\<T\>](#predicatet)
4. [Consumer\<T\>](#consumert)
5. [Supplier\<T\>](#suppliert)
6. [BiFunction and Friends](#bifunction-and-friends)
7. [UnaryOperator and BinaryOperator](#unaryoperator-and-binaryoperator)
8. [Specialized Functional Interfaces](#specialized-functional-interfaces)
9. [Method References](#method-references)
10. [Functional Composition](#functional-composition)
11. [Higher-Order Functions](#higher-order-functions)
12. [Real-World Patterns](#real-world-patterns)
13. [Best Practices](#best-practices)
14. [Common Pitfalls](#common-pitfalls)

---

## Functional Interfaces Overview

### What is a Functional Interface?

```java
/**
 * A functional interface is an interface with exactly ONE abstract method.
 * Can have default methods and static methods.
 * Can be implemented using lambda expressions or method references.
 */

@FunctionalInterface
public interface MyFunction {
    String process(String input);  // Single abstract method (SAM)
    
    // Default methods allowed
    default String processWithLogging(String input) {
        System.out.println("Processing: " + input);
        return process(input);
    }
    
    // Static methods allowed
    static MyFunction identity() {
        return s -> s;
    }
}

// Usage with lambda
MyFunction func = s -> s.toUpperCase();
String result = func.process("hello");  // "HELLO"

// Usage with method reference
MyFunction func2 = String::toUpperCase;
```

### Core Functional Interfaces

```java
import java.util.function.*;

/**
 * Main Functional Interfaces in java.util.function:
 * 
 * 1. Function<T, R>      - Takes T, returns R
 * 2. Predicate<T>        - Takes T, returns boolean
 * 3. Consumer<T>         - Takes T, returns nothing (void)
 * 4. Supplier<T>         - Takes nothing, returns T
 * 5. BiFunction<T, U, R> - Takes T and U, returns R
 * 6. UnaryOperator<T>    - Takes T, returns T (special Function)
 * 7. BinaryOperator<T>   - Takes T and T, returns T (special BiFunction)
 * 
 * + Specialized versions for primitives (IntFunction, IntPredicate, etc.)
 */
```

---

## Function\<T, R\>

### Basic Usage

```java
import java.util.function.Function;

// Function takes one argument and returns a result
Function<String, Integer> stringLength = s -> s.length();
Integer length = stringLength.apply("Hello");  // 5

// Method reference
Function<String, Integer> stringLength2 = String::length;

// Multiple examples
Function<String, String> toUpper = String::toUpperCase;
Function<Integer, Integer> square = x -> x * x;
Function<User, String> getName = User::getName;
Function<String, LocalDate> parseDate = LocalDate::parse;
```

### Composition with andThen

```java
// andThen: Execute this function, THEN the next one
Function<String, String> trim = String::trim;
Function<String, String> toUpper = String::toUpperCase;

Function<String, String> trimAndUpper = trim.andThen(toUpper);
String result = trimAndUpper.apply("  hello  ");  // "HELLO"

// Multiple compositions
Function<String, String> process = String::trim
    .andThen(String::toLowerCase)
    .andThen(s -> s.replace(" ", "-"));

String result2 = process.apply("  Hello World  ");  // "hello-world"

// Real-world example: Data transformation pipeline
Function<String, User> parseJson = JsonParser::parseUser;
Function<User, UserDTO> toDTO = UserMapper::toDTO;
Function<UserDTO, String> serialize = dto -> dto.toString();

Function<String, String> pipeline = parseJson
    .andThen(toDTO)
    .andThen(serialize);

String output = pipeline.apply(jsonInput);
```

### Composition with compose

```java
// compose: Execute the argument function FIRST, then this one
Function<Integer, Integer> multiplyBy2 = x -> x * 2;
Function<Integer, Integer> add3 = x -> x + 3;

// f.compose(g) === f(g(x))
Function<Integer, Integer> add3ThenMultiply = multiplyBy2.compose(add3);
Integer result = add3ThenMultiply.apply(5);  // (5 + 3) * 2 = 16

// Compare with andThen
Function<Integer, Integer> multiplyThenAdd = multiplyBy2.andThen(add3);
Integer result2 = multiplyThenAdd.apply(5);  // (5 * 2) + 3 = 13

// Real-world: Request processing
Function<String, String> validateInput = input -> {
    if (input == null || input.isEmpty()) {
        throw new IllegalArgumentException("Invalid input");
    }
    return input;
};

Function<String, String> sanitize = input -> input.replaceAll("[^a-zA-Z0-9]", "");
Function<String, String> normalize = String::toLowerCase;

Function<String, String> processInput = normalize
    .compose(sanitize)
    .compose(validateInput);  // Validate first, then sanitize, then normalize
```

### Identity Function

```java
// Identity function returns input unchanged
Function<String, String> identity = Function.identity();
String result = identity.apply("hello");  // "hello"

// Useful in streams
List<String> strings = Arrays.asList("a", "b", "c");
Map<String, String> map = strings.stream()
    .collect(Collectors.toMap(
        Function.identity(),  // Key is the string itself
        String::toUpperCase   // Value is uppercase
    ));
// Result: {a=A, b=B, c=C}
```

---

## Predicate\<T\>

### Basic Usage

```java
import java.util.function.Predicate;

// Predicate tests a condition and returns boolean
Predicate<String> isEmpty = String::isEmpty;
boolean result = isEmpty.test("");  // true

Predicate<Integer> isEven = n -> n % 2 == 0;
boolean even = isEven.test(4);  // true

Predicate<User> isAdmin = User::isAdmin;
Predicate<String> hasAtSymbol = s -> s.contains("@");
```

### Logical Operations

```java
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven = n -> n % 2 == 0;

// AND - both must be true
Predicate<Integer> isPositiveAndEven = isPositive.and(isEven);
boolean result1 = isPositiveAndEven.test(4);   // true
boolean result2 = isPositiveAndEven.test(-4);  // false

// OR - at least one must be true
Predicate<Integer> isPositiveOrEven = isPositive.or(isEven);
boolean result3 = isPositiveOrEven.test(-4);  // true (even)
boolean result4 = isPositiveOrEven.test(3);   // true (positive)

// NEGATE - opposite
Predicate<Integer> isNotPositive = isPositive.negate();
boolean result5 = isNotPositive.test(-5);  // true

// Complex combinations
Predicate<Integer> inRange = n -> n >= 10 && n <= 100;
Predicate<Integer> isDivisibleBy5 = n -> n % 5 == 0;

Predicate<Integer> valid = inRange
    .and(isDivisibleBy5)
    .and(isPositive);

boolean result6 = valid.test(50);  // true
```

### Filtering Collections

```java
List<User> users = Arrays.asList(
    new User("John", 25, true),
    new User("Jane", 30, false),
    new User("Bob", 35, true)
);

// Filter with predicates
Predicate<User> isAdmin = User::isAdmin;
Predicate<User> isAdult = u -> u.getAge() >= 18;
Predicate<User> hasLongName = u -> u.getName().length() > 5;

List<User> admins = users.stream()
    .filter(isAdmin)
    .collect(Collectors.toList());

List<User> adultAdmins = users.stream()
    .filter(isAdmin.and(isAdult))
    .collect(Collectors.toList());

// Reusable predicates
public class UserPredicates {
    public static Predicate<User> olderThan(int age) {
        return user -> user.getAge() > age;
    }
    
    public static Predicate<User> hasRole(String role) {
        return user -> user.getRoles().contains(role);
    }
    
    public static Predicate<User> isActive() {
        return User::isActive;
    }
}

// Usage
List<User> seniorAdmins = users.stream()
    .filter(UserPredicates.olderThan(30)
        .and(UserPredicates.hasRole("ADMIN"))
        .and(UserPredicates.isActive()))
    .collect(Collectors.toList());
```

### Predicate Factory Methods

```java
// Static predicate factories
public class Predicates {
    
    public static <T> Predicate<T> isNull() {
        return Objects::isNull;
    }
    
    public static <T> Predicate<T> isNotNull() {
        return Objects::nonNull;
    }
    
    public static <T> Predicate<T> isEqual(T target) {
        return Predicate.isEqual(target);
    }
    
    public static <T> Predicate<T> not(Predicate<T> predicate) {
        return predicate.negate();
    }
    
    // Combine multiple predicates
    @SafeVarargs
    public static <T> Predicate<T> allOf(Predicate<T>... predicates) {
        return Arrays.stream(predicates)
            .reduce(Predicate::and)
            .orElse(t -> true);
    }
    
    @SafeVarargs
    public static <T> Predicate<T> anyOf(Predicate<T>... predicates) {
        return Arrays.stream(predicates)
            .reduce(Predicate::or)
            .orElse(t -> false);
    }
}

// Usage
Predicate<String> isNotEmpty = Predicates.not(String::isEmpty);
Predicate<String> equalsHello = Predicates.isEqual("hello");

Predicate<User> validUser = Predicates.allOf(
    User::isActive,
    u -> u.getAge() >= 18,
    u -> u.getEmail() != null
);
```

---

## Consumer\<T\>

### Basic Usage

```java
import java.util.function.Consumer;

// Consumer accepts input and performs action (no return value)
Consumer<String> print = System.out::println;
print.accept("Hello");  // Prints "Hello"

Consumer<User> sendEmail = user -> emailService.send(user.getEmail());
Consumer<String> log = message -> logger.info(message);

// With lambda
Consumer<List<String>> printAll = list -> list.forEach(System.out::println);
```

### Chaining with andThen

```java
Consumer<User> saveUser = user -> database.save(user);
Consumer<User> logUser = user -> logger.info("Saved: " + user.getName());
Consumer<User> sendEmail = user -> emailService.send(user);

// Chain operations
Consumer<User> saveAndNotify = saveUser
    .andThen(logUser)
    .andThen(sendEmail);

saveAndNotify.accept(newUser);

// Real-world: Order processing
Consumer<Order> validateOrder = order -> validator.validate(order);
Consumer<Order> calculateTotal = order -> order.calculateTotal();
Consumer<Order> applyDiscount = order -> order.applyDiscount();
Consumer<Order> saveOrder = order -> repository.save(order);
Consumer<Order> sendConfirmation = order -> emailService.sendConfirmation(order);

Consumer<Order> processOrder = validateOrder
    .andThen(calculateTotal)
    .andThen(applyDiscount)
    .andThen(saveOrder)
    .andThen(sendConfirmation);

processOrder.accept(order);
```

### forEach and Iteration

```java
List<User> users = getUsers();

// Simple forEach
users.forEach(user -> System.out.println(user.getName()));
users.forEach(System.out::println);

// Complex operations
Consumer<User> updateUser = user -> {
    user.setLastLogin(LocalDateTime.now());
    user.incrementLoginCount();
    repository.save(user);
};

users.forEach(updateUser);

// Conditional execution
Consumer<User> activateIfInactive = user -> {
    if (!user.isActive()) {
        user.setActive(true);
        repository.save(user);
    }
};

users.forEach(activateIfInactive);
```

---

## Supplier\<T\>

### Basic Usage

```java
import java.util.function.Supplier;

// Supplier provides a value (no input)
Supplier<String> getString = () -> "Hello";
String value = getString.get();  // "Hello"

Supplier<Double> random = Math::random;
Supplier<LocalDateTime> now = LocalDateTime::now;
Supplier<User> createUser = User::new;

// Lazy evaluation
Supplier<ExpensiveObject> lazyInit = () -> new ExpensiveObject();
// Object not created until get() is called
ExpensiveObject obj = lazyInit.get();
```

### Factory Pattern

```java
// Supplier as factory
public class UserFactory {
    
    private static final Supplier<User> GUEST_USER = () -> 
        new User("guest", "guest@example.com");
    
    private static final Supplier<User> ADMIN_USER = () -> 
        new User("admin", "admin@example.com", Role.ADMIN);
    
    public static User createGuest() {
        return GUEST_USER.get();
    }
    
    public static User createAdmin() {
        return ADMIN_USER.get();
    }
    
    // Parameterized factory
    public static Supplier<User> withEmail(String email) {
        return () -> new User(UUID.randomUUID().toString(), email);
    }
}

// Usage
User guest = UserFactory.createGuest();
User admin = UserFactory.createAdmin();
User custom = UserFactory.withEmail("user@example.com").get();
```

### Lazy Initialization

```java
public class LazyValue<T> {
    private Supplier<T> supplier;
    private T value;
    
    public LazyValue(Supplier<T> supplier) {
        this.supplier = supplier;
    }
    
    public T get() {
        if (value == null) {
            value = supplier.get();
            supplier = null;  // Allow GC
        }
        return value;
    }
}

// Usage
LazyValue<ExpensiveResource> resource = 
    new LazyValue<>(() -> new ExpensiveResource());

// Resource not created yet
// ...

// Created on first access
ExpensiveResource res = resource.get();
```

### Default Values and Optional

```java
// Supplier with Optional.orElseGet
Optional<String> optional = Optional.empty();
String value = optional.orElseGet(() -> "Default Value");

// Avoid expensive operations
String value2 = optional.orElseGet(() -> {
    // Complex computation only if optional is empty
    return computeExpensiveDefault();
});

// Compare with orElse (always evaluates)
String value3 = optional.orElse(computeExpensiveDefault());  // Always called!

// Supplier for random values
public class Defaults {
    public static Supplier<String> randomId() {
        return () -> UUID.randomUUID().toString();
    }
    
    public static Supplier<LocalDateTime> currentTime() {
        return LocalDateTime::now;
    }
    
    public static <T> Supplier<List<T>> emptyList() {
        return ArrayList::new;
    }
}

// Usage
String id = Defaults.randomId().get();
LocalDateTime time = Defaults.currentTime().get();
List<String> list = Defaults.emptyList().get();
```

---

## BiFunction and Friends

### BiFunction\<T, U, R\>

```java
import java.util.function.BiFunction;

// BiFunction takes two arguments and returns a result
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
Integer sum = add.apply(5, 3);  // 8

BiFunction<String, String, String> concat = (s1, s2) -> s1 + s2;
BiFunction<User, String, User> updateEmail = (user, email) -> {
    user.setEmail(email);
    return user;
};

// Real-world examples
BiFunction<Double, Double, Double> calculateDiscount = (price, discountPercent) -> 
    price * (1 - discountPercent / 100);

BiFunction<String, String, User> createUser = (name, email) -> 
    new User(UUID.randomUUID().toString(), name, email);

// With andThen
BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
Function<Integer, String> toString = Object::toString;

BiFunction<Integer, Integer, String> multiplyAndConvert = multiply.andThen(toString);
String result = multiplyAndConvert.apply(5, 3);  // "15"
```

### BiConsumer\<T, U\>

```java
import java.util.function.BiConsumer;

// BiConsumer accepts two arguments, returns nothing
BiConsumer<String, String> printPair = (key, value) -> 
    System.out.println(key + " = " + value);

printPair.accept("name", "John");  // name = John

// Map forEach
Map<String, Integer> map = Map.of("a", 1, "b", 2, "c", 3);
map.forEach((key, value) -> System.out.println(key + ": " + value));

// Real-world: Logging
BiConsumer<String, Throwable> logError = (message, error) -> 
    logger.error(message, error);

BiConsumer<User, String> sendNotification = (user, message) -> 
    emailService.send(user.getEmail(), message);

// Chaining
BiConsumer<User, Order> processOrder = (user, order) -> 
    orderService.process(order);

BiConsumer<User, Order> sendConfirmation = (user, order) -> 
    emailService.sendConfirmation(user, order);

BiConsumer<User, Order> processAndNotify = processOrder.andThen(sendConfirmation);
processAndNotify.accept(user, order);
```

### BiPredicate\<T, U\>

```java
import java.util.function.BiPredicate;

// BiPredicate tests two arguments
BiPredicate<String, String> equals = String::equals;
boolean result = equals.test("hello", "hello");  // true

BiPredicate<Integer, Integer> isGreaterThan = (a, b) -> a > b;
BiPredicate<User, String> hasRole = (user, role) -> 
    user.getRoles().contains(role);

// With logical operations
BiPredicate<String, Integer> isValidPassword = (password, minLength) -> 
    password != null && password.length() >= minLength;

BiPredicate<String, Integer> containsNumber = (password, minNumbers) -> 
    password.chars().filter(Character::isDigit).count() >= minNumbers;

BiPredicate<String, Integer> isStrongPassword = 
    isValidPassword.and(containsNumber);

boolean strong = isStrongPassword.test("pass123", 8);
```

---

## UnaryOperator and BinaryOperator

### UnaryOperator\<T\>

```java
import java.util.function.UnaryOperator;

// UnaryOperator is Function<T, T> - same input/output type
UnaryOperator<String> toUpper = String::toUpperCase;
String result = toUpper.apply("hello");  // "HELLO"

UnaryOperator<Integer> square = x -> x * x;
UnaryOperator<Double> increment = x -> x + 1;

// List.replaceAll uses UnaryOperator
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
list.replaceAll(String::toUpperCase);  // ["A", "B", "C"]

List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
numbers.replaceAll(n -> n * 2);  // [2, 4, 6, 8, 10]

// Identity
UnaryOperator<String> identity = UnaryOperator.identity();
String same = identity.apply("hello");  // "hello"

// Composition
UnaryOperator<String> trim = String::trim;
UnaryOperator<String> upper = String::toUpperCase;
UnaryOperator<String> trimAndUpper = trim.andThen(upper);

String result2 = trimAndUpper.apply("  hello  ");  // "HELLO"
```

### BinaryOperator\<T\>

```java
import java.util.function.BinaryOperator;

// BinaryOperator is BiFunction<T, T, T> - same types for all
BinaryOperator<Integer> add = (a, b) -> a + b;
BinaryOperator<Integer> max = Math::max;
BinaryOperator<String> concat = (s1, s2) -> s1 + s2;

// Stream reduce
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
Integer sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);  // BinaryOperator

Integer max = numbers.stream()
    .reduce(Integer::max)
    .orElse(0);

// maxBy and minBy
BinaryOperator<User> youngestUser = BinaryOperator.minBy(
    Comparator.comparing(User::getAge)
);

BinaryOperator<User> oldestUser = BinaryOperator.maxBy(
    Comparator.comparing(User::getAge)
);

List<User> users = getUsers();
User youngest = users.stream()
    .reduce(youngestUser)
    .orElse(null);

// Merging
Map<String, Integer> map1 = Map.of("a", 1, "b", 2);
Map<String, Integer> map2 = Map.of("b", 3, "c", 4);

BinaryOperator<Integer> mergeFunction = (v1, v2) -> v1 + v2;

Map<String, Integer> merged = Stream.of(map1, map2)
    .flatMap(m -> m.entrySet().stream())
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        Map.Entry::getValue,
        mergeFunction  // BinaryOperator for conflicts
    ));
// Result: {a=1, b=5, c=4}
```

---

## Specialized Functional Interfaces

### Primitive Functional Interfaces

```java
// IntFunction, IntPredicate, IntConsumer, IntSupplier
import java.util.function.*;

// IntFunction<R> - int → R
IntFunction<String> intToString = i -> "Number: " + i;
String result = intToString.apply(42);

// IntPredicate - int → boolean
IntPredicate isEven = i -> i % 2 == 0;
boolean even = isEven.test(4);

// IntConsumer - int → void
IntConsumer printInt = System.out::println;
printInt.accept(42);

// IntSupplier - () → int
IntSupplier random = () -> ThreadLocalRandom.current().nextInt(100);
int value = random.getAsInt();

// IntUnaryOperator - int → int
IntUnaryOperator square = i -> i * i;
int squared = square.applyAsInt(5);

// IntBinaryOperator - (int, int) → int
IntBinaryOperator add = (a, b) -> a + b;
int sum = add.applyAsInt(5, 3);

// Similar for Long and Double
LongFunction<String> longToString = l -> "Long: " + l;
DoublePredicate isPositive = d -> d > 0;
DoubleConsumer printDouble = System.out::println;
DoubleSupplier randomDouble = Math::random;

// Avoid boxing/unboxing overhead
IntStream.range(0, 1000)
    .filter(isEven)           // IntPredicate
    .map(square)              // IntUnaryOperator
    .forEach(printInt);       // IntConsumer
```

### ToIntFunction, ToLongFunction, ToDoubleFunction

```java
// Function<T, R> specialized for primitive returns

ToIntFunction<String> stringLength = String::length;
int length = stringLength.applyAsInt("Hello");

ToDoubleFunction<User> getUserAge = User::getAge;
double age = getUserAge.applyAsDouble(user);

// With streams
List<String> strings = Arrays.asList("a", "bb", "ccc");
int totalLength = strings.stream()
    .mapToInt(String::length)  // Uses ToIntFunction
    .sum();

List<User> users = getUsers();
double averageAge = users.stream()
    .mapToDouble(User::getAge)  // Uses ToDoubleFunction
    .average()
    .orElse(0.0);
```

---

## Method References

### Types of Method References

```java
// 1. Static method reference
Function<String, Integer> parseInt = Integer::parseInt;
int number = parseInt.apply("123");

// 2. Instance method of particular object
String prefix = "Hello, ";
Function<String, String> addPrefix = prefix::concat;
String result = addPrefix.apply("World");  // "Hello, World"

// 3. Instance method of arbitrary object of particular type
Function<String, String> toUpper = String::toUpperCase;
String upper = toUpper.apply("hello");

// 4. Constructor reference
Supplier<User> userSupplier = User::new;
User user = userSupplier.get();

Function<String, User> userWithId = User::new;  // Constructor with parameter
User user2 = userWithId.apply("123");

// Comparison: Lambda vs Method Reference
// Lambda
Function<String, Integer> f1 = s -> s.length();
// Method reference (preferred when possible)
Function<String, Integer> f2 = String::length;

// Lambda
Consumer<String> c1 = s -> System.out.println(s);
// Method reference
Consumer<String> c2 = System.out::println;

// Lambda
Supplier<List<String>> s1 = () -> new ArrayList<>();
// Method reference
Supplier<List<String>> s2 = ArrayList::new;
```

### Real-World Method References

```java
// Stream operations
List<String> names = users.stream()
    .map(User::getName)          // Method reference
    .collect(Collectors.toList());

List<String> uppercased = names.stream()
    .map(String::toUpperCase)    // Method reference
    .collect(Collectors.toList());

// Filtering
List<User> admins = users.stream()
    .filter(User::isAdmin)       // Method reference
    .collect(Collectors.toList());

// Sorting
List<User> sorted = users.stream()
    .sorted(Comparator.comparing(User::getAge))  // Method reference
    .collect(Collectors.toList());

// forEach
users.forEach(System.out::println);        // Method reference
users.forEach(emailService::send);         // Method reference
```

---

## Functional Composition

### Composing Functions

```java
// Building complex operations from simple ones
Function<String, String> trim = String::trim;
Function<String, String> toLowerCase = String::toLowerCase;
Function<String, String> removeDashes = s -> s.replace("-", "");

Function<String, String> normalizeInput = trim
    .andThen(toLowerCase)
    .andThen(removeDashes);

String result = normalizeInput.apply("  HELLO-WORLD  ");  // "helloworld"

// Validation pipeline
Function<String, String> validateNotNull = s -> {
    if (s == null) throw new IllegalArgumentException("Null input");
    return s;
};

Function<String, String> validateNotEmpty = s -> {
    if (s.isEmpty()) throw new IllegalArgumentException("Empty input");
    return s;
};

Function<String, String> validateEmail = s -> {
    if (!s.contains("@")) throw new IllegalArgumentException("Invalid email");
    return s;
};

Function<String, String> emailValidator = validateNotNull
    .andThen(validateNotEmpty)
    .andThen(trim)
    .andThen(validateEmail);

String validEmail = emailValidator.apply("user@example.com");
```

### Composing Predicates

```java
// Complex validation with predicate composition
Predicate<String> hasMinLength = s -> s.length() >= 8;
Predicate<String> hasUpperCase = s -> !s.equals(s.toLowerCase());
Predicate<String> hasLowerCase = s -> !s.equals(s.toUpperCase());
Predicate<String> hasDigit = s -> s.matches(".*\\d.*");
Predicate<String> hasSpecialChar = s -> s.matches(".*[!@#$%^&*].*");

Predicate<String> isStrongPassword = hasMinLength
    .and(hasUpperCase)
    .and(hasLowerCase)
    .and(hasDigit)
    .and(hasSpecialChar);

boolean strong = isStrongPassword.test("MyP@ss123");

// Or simplified
Predicate<String> isWeakPassword = isStrongPassword.negate();
```

### Composing Consumers

```java
// Multi-step processing
Consumer<User> validateUser = user -> validator.validate(user);
Consumer<User> saveUser = user -> repository.save(user);
Consumer<User> indexUser = user -> searchEngine.index(user);
Consumer<User> sendWelcomeEmail = user -> emailService.sendWelcome(user);
Consumer<User> logCreation = user -> logger.info("Created: " + user.getId());

Consumer<User> createUserPipeline = validateUser
    .andThen(saveUser)
    .andThen(indexUser)
    .andThen(sendWelcomeEmail)
    .andThen(logCreation);

createUserPipeline.accept(newUser);
```

---

## Higher-Order Functions

### Functions that Return Functions

```java
// Function factory
public class Functions {
    
    // Returns a function that adds a specific value
    public static Function<Integer, Integer> adder(int value) {
        return n -> n + value;
    }
    
    // Returns a function that multiplies by a specific value
    public static Function<Integer, Integer> multiplier(int factor) {
        return n -> n * factor;
    }
    
    // Returns a predicate that checks if value is greater than threshold
    public static Predicate<Integer> greaterThan(int threshold) {
        return n -> n > threshold;
    }
    
    // Returns a consumer that logs with a specific prefix
    public static <T> Consumer<T> logger(String prefix) {
        return value -> System.out.println(prefix + ": " + value);
    }
}

// Usage
Function<Integer, Integer> add10 = Functions.adder(10);
int result = add10.apply(5);  // 15

Predicate<Integer> isAdult = Functions.greaterThan(18);
boolean adult = isAdult.test(25);  // true

Consumer<String> errorLogger = Functions.logger("ERROR");
errorLogger.accept("Something went wrong");  // ERROR: Something went wrong
```

### Functions that Accept Functions

```java
public class Executor {
    
    // Execute function with logging
    public static <T, R> Function<T, R> withLogging(
            Function<T, R> function,
            String description) {
        
        return input -> {
            System.out.println("Executing: " + description);
            R result = function.apply(input);
            System.out.println("Result: " + result);
            return result;
        };
    }
    
    // Execute function with retry
    public static <T, R> Function<T, R> withRetry(
            Function<T, R> function,
            int maxAttempts) {
        
        return input -> {
            for (int i = 0; i < maxAttempts; i++) {
                try {
                    return function.apply(input);
                } catch (Exception e) {
                    if (i == maxAttempts - 1) {
                        throw e;
                    }
                    System.out.println("Retry attempt " + (i + 1));
                }
            }
            throw new RuntimeException("All retries failed");
        };
    }
    
    // Execute function with timing
    public static <T, R> Function<T, R> withTiming(Function<T, R> function) {
        return input -> {
            long start = System.currentTimeMillis();
            R result = function.apply(input);
            long duration = System.currentTimeMillis() - start;
            System.out.println("Execution time: " + duration + "ms");
            return result;
        };
    }
}

// Usage
Function<String, String> parseJson = JsonParser::parse;

Function<String, String> resilientParser = Executor.withLogging(
    Executor.withRetry(
        Executor.withTiming(parseJson),
        3
    ),
    "JSON Parsing"
);

String result = resilientParser.apply(jsonString);
```

---

## Real-World Patterns

### Strategy Pattern

```java
public interface PaymentProcessor {
    void process(Payment payment);
}

// Traditional approach
public class CreditCardProcessor implements PaymentProcessor {
    public void process(Payment payment) { /* ... */ }
}

// Functional approach
public class PaymentService {
    
    private final Map<PaymentType, Consumer<Payment>> processors = Map.of(
        PaymentType.CREDIT_CARD, this::processCreditCard,
        PaymentType.PAYPAL, this::processPayPal,
        PaymentType.BANK_TRANSFER, this::processBankTransfer
    );
    
    public void process(Payment payment) {
        processors.get(payment.getType()).accept(payment);
    }
    
    private void processCreditCard(Payment payment) { /* ... */ }
    private void processPayPal(Payment payment) { /* ... */ }
    private void processBankTransfer(Payment payment) { /* ... */ }
}
```

### Template Method Pattern

```java
public class DataProcessor<T> {
    
    private final Function<String, T> parser;
    private final Predicate<T> validator;
    private final Consumer<T> processor;
    private final Consumer<Exception> errorHandler;
    
    public DataProcessor(
            Function<String, T> parser,
            Predicate<T> validator,
            Consumer<T> processor,
            Consumer<Exception> errorHandler) {
        
        this.parser = parser;
        this.validator = validator;
        this.processor = processor;
        this.errorHandler = errorHandler;
    }
    
    public void process(String data) {
        try {
            T parsed = parser.apply(data);
            
            if (validator.test(parsed)) {
                processor.accept(parsed);
            } else {
                throw new ValidationException("Invalid data");
            }
        } catch (Exception e) {
            errorHandler.accept(e);
        }
    }
}

// Usage
DataProcessor<User> userProcessor = new DataProcessor<>(
    JsonParser::parseUser,        // Parser
    User::isValid,                 // Validator
    repository::save,              // Processor
    error -> logger.error("Error", error)  // Error handler
);

userProcessor.process(jsonData);
```

### Chain of Responsibility

```java
public class ValidationChain {
    
    private final List<Predicate<User>> validators;
    
    public ValidationChain(Predicate<User>... validators) {
        this.validators = Arrays.asList(validators);
    }
    
    public boolean validate(User user) {
        return validators.stream()
            .allMatch(validator -> validator.test(user));
    }
    
    public List<String> getErrors(User user) {
        return validators.stream()
            .filter(v -> !v.test(user))
            .map(Object::toString)
            .collect(Collectors.toList());
    }
}

// Usage
ValidationChain userValidation = new ValidationChain(
    user -> user.getName() != null,
    user -> user.getEmail() != null && user.getEmail().contains("@"),
    user -> user.getAge() >= 18,
    user -> !user.isBanned()
);

boolean valid = userValidation.validate(user);
```

---

## Best Practices

### ✅ DO

1. **Prefer method references over lambdas when possible**
   ```java
   // ✅ GOOD
   list.forEach(System.out::println);
   
   // ❌ VERBOSE
   list.forEach(s -> System.out.println(s));
   ```

2. **Use specialized functional interfaces for primitives**
   ```java
   // ✅ GOOD - no boxing
   IntPredicate isEven = i -> i % 2 == 0;
   
   // ❌ BAD - boxing overhead
   Predicate<Integer> isEven = i -> i % 2 == 0;
   ```

3. **Compose functions for readability**
   ```java
   // ✅ GOOD - clear pipeline
   Function<String, String> normalize = String::trim
       .andThen(String::toLowerCase)
       .andThen(s -> s.replace(" ", "-"));
   
   // ❌ BAD - nested calls
   String result = input.trim().toLowerCase().replace(" ", "-");
   ```

4. **Use Supplier for lazy evaluation**
   ```java
   // ✅ GOOD - lazy
   String value = optional.orElseGet(() -> expensiveOperation());
   
   // ❌ BAD - always evaluates
   String value = optional.orElse(expensiveOperation());
   ```

5. **Extract complex lambdas to methods**
   ```java
   // ✅ GOOD - readable
   list.stream()
       .filter(this::isValidUser)
       .map(this::toDto)
       .collect(Collectors.toList());
   
   // ❌ BAD - complex inline lambdas
   list.stream()
       .filter(u -> u.getAge() >= 18 && u.isActive() && !u.isBanned())
       .map(u -> new UserDto(u.getId(), u.getName()))
       .collect(Collectors.toList());
   ```

6. **Use function composition for pipelines**
7. **Document complex functional code**
8. **Prefer immutability in functional operations**
9. **Use Optional with functional methods**
10. **Keep lambdas short and focused**

### ❌ DON'T

1. **Don't modify external state in functions**
2. **Don't ignore exceptions in lambdas**
3. **Don't use null in functional code**
4. **Don't create complex nested lambdas**
5. **Don't forget about readability**

---

## Conclusion

**Key Takeaways:**

1. **Function<T, R>** - Transformation
2. **Predicate<T>** - Testing/filtering
3. **Consumer<T>** - Side effects
4. **Supplier<T>** - Lazy value generation
5. **Composition** - Build complex operations from simple ones
6. **Method References** - Cleaner than lambdas when possible
7. **Higher-Order Functions** - Functions that work with functions
8. **Patterns** - Strategy, template, chain of responsibility

**Remember**: Functional interfaces enable declarative, composable, and testable code. Use them to write cleaner, more maintainable Java!
