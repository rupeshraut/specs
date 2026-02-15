# Effective Java Generics with Modern Java

A comprehensive guide to mastering Java generics from fundamentals to advanced patterns, including modern Java features and production-ready best practices.

---

## Table of Contents

1. [Generics Fundamentals](#generics-fundamentals)
2. [Type Parameters and Bounds](#type-parameters-and-bounds)
3. [Wildcards Deep Dive](#wildcards-deep-dive)
4. [PECS Principle](#pecs-principle)
5. [Generic Methods](#generic-methods)
6. [Type Erasure](#type-erasure)
7. [Generics with Records](#generics-with-records)
8. [Generics with Sealed Types](#generics-with-sealed-types)
9. [Generics and Pattern Matching](#generics-and-pattern-matching)
10. [Advanced Patterns](#advanced-patterns)
11. [Common Pitfalls](#common-pitfalls)
12. [Best Practices](#best-practices)

---

## Generics Fundamentals

### Why Generics?

```java
// ❌ Before Generics (pre-Java 5)
List list = new ArrayList();
list.add("String");
list.add(42); // Oops! No type safety

String str = (String) list.get(0); // Explicit cast required
String num = (String) list.get(1); // ClassCastException at runtime!

// ✅ With Generics (Java 5+)
List<String> list = new ArrayList<>();
list.add("String");
// list.add(42); // Compile-time error - type safety!

String str = list.get(0); // No cast needed
```

### Generic Classes

```java
// Basic generic class
public class Box<T> {
    private T value;

    public Box(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }

    public void setValue(T value) {
        this.value = value;
    }
}

// Usage
Box<String> stringBox = new Box<>("Hello");
Box<Integer> intBox = new Box<>(42);

String str = stringBox.getValue(); // No cast needed
Integer num = intBox.getValue();

// Multiple type parameters
public class Pair<K, V> {
    private final K key;
    private final V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }
}

// Usage
Pair<String, Integer> pair = new Pair<>("age", 30);
```

### Generic Interfaces

```java
// Generic interface
public interface Repository<T, ID> {
    T save(T entity);
    Optional<T> findById(ID id);
    List<T> findAll();
    void delete(T entity);
}

// Implementation
public class UserRepository implements Repository<User, Long> {

    @Override
    public User save(User entity) {
        // Save user to database
        return entity;
    }

    @Override
    public Optional<User> findById(Long id) {
        // Find user by ID
        return Optional.empty();
    }

    @Override
    public List<User> findAll() {
        // Return all users
        return List.of();
    }

    @Override
    public void delete(User entity) {
        // Delete user
    }
}
```

---

## Type Parameters and Bounds

### Upper Bounds (extends)

```java
// Upper bound - T must be Number or subclass
public class NumberBox<T extends Number> {
    private T value;

    public NumberBox(T value) {
        this.value = value;
    }

    // Can use Number methods
    public double getDoubleValue() {
        return value.doubleValue();
    }

    public int compareTo(T other) {
        return Double.compare(
            value.doubleValue(),
            other.doubleValue()
        );
    }
}

// Usage
NumberBox<Integer> intBox = new NumberBox<>(42);
NumberBox<Double> doubleBox = new NumberBox<>(3.14);
// NumberBox<String> stringBox = new NumberBox<>("nope"); // Compile error!

// Multiple bounds
public class ComparableBox<T extends Number & Comparable<T>> {
    private T value;

    public ComparableBox(T value) {
        this.value = value;
    }

    public boolean isGreaterThan(T other) {
        // Can use both Number and Comparable methods
        return value.compareTo(other) > 0;
    }

    public double getAsDouble() {
        return value.doubleValue();
    }
}

// Usage
ComparableBox<Integer> box = new ComparableBox<>(42);
boolean result = box.isGreaterThan(30); // true
```

### Real-World Example: Builder with Bounds

```java
// Generic builder with type bounds
public class QueryBuilder<T extends BaseEntity> {
    private final Class<T> entityClass;
    private final List<Predicate> predicates = new ArrayList<>();
    private Integer limit;
    private Integer offset;

    public QueryBuilder(Class<T> entityClass) {
        this.entityClass = entityClass;
    }

    public QueryBuilder<T> where(String field, Object value) {
        predicates.add(new Predicate(field, value));
        return this;
    }

    public QueryBuilder<T> limit(int limit) {
        this.limit = limit;
        return this;
    }

    public QueryBuilder<T> offset(int offset) {
        this.offset = offset;
        return this;
    }

    public List<T> execute() {
        // Build and execute query
        return List.of();
    }
}

// Usage
List<User> users = new QueryBuilder<>(User.class)
    .where("age", 30)
    .where("status", "ACTIVE")
    .limit(10)
    .execute();
```

---

## Wildcards Deep Dive

### Unbounded Wildcard (?)

```java
// Unbounded wildcard - any type
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Works with any list
printList(List.of(1, 2, 3));
printList(List.of("a", "b", "c"));

// ❌ Cannot add elements (except null)
public void cannotAdd(List<?> list) {
    // list.add("string"); // Compile error!
    // list.add(42);       // Compile error!
    list.add(null);        // OK but useless
}
```

### Upper Bounded Wildcard (? extends T)

```java
// Producer - reading from collection
public double sumNumbers(List<? extends Number> numbers) {
    double sum = 0;
    for (Number num : numbers) {
        sum += num.doubleValue();
    }
    return sum;
}

// Works with Number and all subtypes
sumNumbers(List.of(1, 2, 3));           // List<Integer>
sumNumbers(List.of(1.5, 2.5, 3.5));     // List<Double>
sumNumbers(List.of(1L, 2L, 3L));        // List<Long>

// ❌ Cannot add elements (except null)
public void cannotAdd(List<? extends Number> numbers) {
    // numbers.add(42);      // Compile error!
    // numbers.add(3.14);    // Compile error!

    // Why? We don't know the exact type
    // Could be List<Integer>, List<Double>, etc.
}

// ✅ Can read as Number or supertype
public Number getFirst(List<? extends Number> numbers) {
    if (!numbers.isEmpty()) {
        return numbers.get(0); // OK - returns Number
    }
    return null;
}
```

### Lower Bounded Wildcard (? super T)

```java
// Consumer - writing to collection
public void addIntegers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    list.add(3);
}

// Works with Integer and all supertypes
addIntegers(new ArrayList<Integer>());    // List<Integer>
addIntegers(new ArrayList<Number>());     // List<Number>
addIntegers(new ArrayList<Object>());     // List<Object>

// ✅ Can add Integer or subtypes
public void add(List<? super Integer> list, Integer value) {
    list.add(value);     // OK
    list.add(42);        // OK
}

// ❌ Cannot safely read as specific type
public void cannotRead(List<? super Integer> list) {
    Object obj = list.get(0);     // OK - can read as Object
    // Integer num = list.get(0);  // Compile error!

    // Why? Could be List<Number>, List<Object>, etc.
    // Element might not be Integer
}
```

---

## PECS Principle

### Producer Extends, Consumer Super

```java
/**
 * PECS Rule:
 * - Producer Extends: Use <? extends T> when you only READ from structure
 * - Consumer Super: Use <? super T> when you only WRITE to structure
 */

// Producer Extends - reading from source
public static <T> void copy(
        List<? extends T> source,    // Producer - extends
        List<? super T> destination  // Consumer - super
) {
    for (T item : source) {
        destination.add(item);
    }
}

// Usage
List<Integer> integers = List.of(1, 2, 3);
List<Number> numbers = new ArrayList<>();

copy(integers, numbers); // Works!

// Real-world example: Collections.copy()
// public static <T> void copy(
//     List<? super T> dest,
//     List<? extends T> src
// )
```

### Practical Examples

```java
// Producer: Reading from collection
public class Statistics {

    // Producer - extends
    public static double average(List<? extends Number> numbers) {
        return numbers.stream()
            .mapToDouble(Number::doubleValue)
            .average()
            .orElse(0.0);
    }

    // Producer - extends
    public static <T extends Comparable<T>> T max(List<? extends T> list) {
        return list.stream()
            .max(Comparator.naturalOrder())
            .orElseThrow();
    }
}

// Consumer: Writing to collection
public class CollectionUtils {

    // Consumer - super
    public static void addNumbers(List<? super Integer> list, int count) {
        for (int i = 0; i < count; i++) {
            list.add(i);
        }
    }

    // Consumer - super
    public static <T> void fill(List<? super T> list, T value, int count) {
        for (int i = 0; i < count; i++) {
            list.add(value);
        }
    }
}

// Usage
List<Number> numbers = new ArrayList<>();
CollectionUtils.addNumbers(numbers, 5); // Adds 0,1,2,3,4
CollectionUtils.fill(numbers, 42, 3);   // Adds 42,42,42

List<Integer> integers = List.of(1, 2, 3, 4, 5);
double avg = Statistics.average(integers); // 3.0
Integer max = Statistics.max(integers);    // 5
```

---

## Generic Methods

### Basic Generic Methods

```java
public class GenericMethods {

    // Generic method - type parameter before return type
    public static <T> T firstElement(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }
        return list.get(0);
    }

    // Multiple type parameters
    public static <K, V> Map<K, V> createMap(K[] keys, V[] values) {
        if (keys.length != values.length) {
            throw new IllegalArgumentException("Arrays must have same length");
        }

        Map<K, V> map = new HashMap<>();
        for (int i = 0; i < keys.length; i++) {
            map.put(keys[i], values[i]);
        }
        return map;
    }

    // Bounded type parameter
    public static <T extends Comparable<T>> T findMax(T a, T b) {
        return a.compareTo(b) > 0 ? a : b;
    }

    // Usage
    public static void main(String[] args) {
        String first = firstElement(List.of("a", "b", "c")); // "a"

        String[] keys = {"name", "age"};
        Object[] values = {"John", 30};
        Map<String, Object> map = createMap(keys, values);

        Integer max = findMax(10, 20); // 20
    }
}
```

### Advanced Generic Methods

```java
public class AdvancedGenericMethods {

    // Generic method with multiple bounds
    public static <T extends Comparable<T> & Serializable> T cloneAndCompare(
            T original, T copy) {

        // Can use methods from both Comparable and Serializable
        int comparison = original.compareTo(copy);
        return comparison > 0 ? original : copy;
    }

    // Generic method with wildcard parameters
    public static <T> void swap(List<T> list, int i, int j) {
        T temp = list.get(i);
        list.set(i, list.get(j));
        list.set(j, temp);
    }

    // Recursive type bound
    public static <T extends Comparable<? super T>> T max(Collection<T> collection) {
        return collection.stream()
            .max(Comparator.naturalOrder())
            .orElseThrow();
    }

    // Generic method returning generic type
    public static <T> Optional<T> findFirst(List<T> list, Predicate<T> predicate) {
        return list.stream()
            .filter(predicate)
            .findFirst();
    }
}
```

---

## Type Erasure

### Understanding Type Erasure

```java
// What you write:
public class Box<T> {
    private T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}

// What compiler generates (after type erasure):
public class Box {
    private Object value; // T becomes Object

    public void set(Object value) {
        this.value = value;
    }

    public Object get() {
        return value;
    }
}

// With bounds:
public class NumberBox<T extends Number> {
    private T value;
}

// After erasure:
public class NumberBox {
    private Number value; // T becomes Number (the bound)
}
```

### Type Erasure Implications

```java
// ❌ Cannot create instances of type parameter
public class Box<T> {
    public T createInstance() {
        // return new T(); // Compile error!
        return null;
    }
}

// ✅ Solution: Pass class object
public class Box<T> {
    private final Class<T> type;

    public Box(Class<T> type) {
        this.type = type;
    }

    public T createInstance() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}

// ❌ Cannot create generic arrays
public class Container<T> {
    public void method() {
        // T[] array = new T[10]; // Compile error!
    }
}

// ✅ Solution: Use List or create array with reflection
public class Container<T> {
    private final Class<T> type;

    public Container(Class<T> type) {
        this.type = type;
    }

    @SuppressWarnings("unchecked")
    public T[] createArray(int size) {
        return (T[]) Array.newInstance(type, size);
    }
}

// ❌ Cannot use instanceof with generic type
public <T> boolean isInstance(Object obj) {
    // if (obj instanceof T) { } // Compile error!
    // if (obj instanceof List<String>) { } // Compile error!

    if (obj instanceof List<?>) { } // OK
    return false;
}
```

### Bridge Methods

```java
// Generic class
public class Node<T> {
    private T data;

    public void setData(T data) {
        this.data = data;
    }
}

// Subclass
public class IntegerNode extends Node<Integer> {
    @Override
    public void setData(Integer data) {
        super.setData(data);
    }
}

// After type erasure, compiler generates bridge method:
public class IntegerNode extends Node {
    // Bridge method (synthetic)
    public void setData(Object data) {
        setData((Integer) data); // Cast and call
    }

    // Your method
    public void setData(Integer data) {
        super.setData(data);
    }
}
```

---

## Generics with Records

### Generic Records (Java 16+)

```java
// Simple generic record
public record Result<T>(T value, String message, boolean success) {

    public static <T> Result<T> success(T value) {
        return new Result<>(value, "Success", true);
    }

    public static <T> Result<T> failure(String message) {
        return new Result<>(null, message, false);
    }
}

// Usage
Result<User> userResult = Result.success(new User("John"));
Result<Payment> failedPayment = Result.failure("Payment declined");

// Generic record with bounds
public record Wrapper<T extends Comparable<T>>(T value) {

    public boolean isGreaterThan(T other) {
        return value.compareTo(other) > 0;
    }

    public T max(T other) {
        return value.compareTo(other) > 0 ? value : other;
    }
}

// Usage
Wrapper<Integer> wrapper = new Wrapper<>(42);
boolean greater = wrapper.isGreaterThan(30); // true
Integer max = wrapper.max(50); // 50
```

### Either Pattern with Records

```java
// Sealed interface with generic records
public sealed interface Either<L, R> {

    record Left<L, R>(L value) implements Either<L, R> {
        public <T> T fold(Function<L, T> leftFn, Function<R, T> rightFn) {
            return leftFn.apply(value);
        }
    }

    record Right<L, R>(R value) implements Either<L, R> {
        public <T> T fold(Function<L, T> leftFn, Function<R, T> rightFn) {
            return rightFn.apply(value);
        }
    }

    static <L, R> Either<L, R> left(L value) {
        return new Left<>(value);
    }

    static <L, R> Either<L, R> right(R value) {
        return new Right<>(value);
    }

    <T> T fold(Function<L, T> leftFn, Function<R, T> rightFn);
}

// Usage - representing success/failure
public Either<String, User> findUser(Long id) {
    try {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("User not found"));
        return Either.right(user);
    } catch (Exception e) {
        return Either.left(e.getMessage());
    }
}

// Pattern matching with Either
Either<String, User> result = findUser(123L);

String message = result.fold(
    error -> "Error: " + error,
    user -> "Found user: " + user.getName()
);
```

---

## Generics with Sealed Types

### Sealed Generic Hierarchies (Java 17+)

```java
// Sealed generic interface
public sealed interface Response<T> {

    record Success<T>(T data) implements Response<T> {}

    record Error<T>(String message, int code) implements Response<T> {}

    record Loading<T>() implements Response<T> {}
}

// Pattern matching with sealed generics
public <T> String handleResponse(Response<T> response) {
    return switch (response) {
        case Response.Success<T>(var data) -> "Success: " + data;
        case Response.Error<T>(var msg, var code) -> "Error " + code + ": " + msg;
        case Response.Loading<T>() -> "Loading...";
    };
}

// Usage
Response<User> userResponse = new Response.Success<>(new User("John"));
String result = handleResponse(userResponse); // "Success: User[name=John]"
```

### Generic Sealed Classes

```java
// Sealed class hierarchy with generics
public sealed abstract class Tree<T> {

    public static final class Leaf<T> extends Tree<T> {
        private final T value;

        public Leaf(T value) {
            this.value = value;
        }

        public T getValue() {
            return value;
        }
    }

    public static final class Node<T> extends Tree<T> {
        private final Tree<T> left;
        private final Tree<T> right;

        public Node(Tree<T> left, Tree<T> right) {
            this.left = left;
            this.right = right;
        }

        public Tree<T> getLeft() {
            return left;
        }

        public Tree<T> getRight() {
            return right;
        }
    }

    // Pattern matching for tree traversal
    public <R> R match(
            Function<T, R> leafFn,
            BiFunction<Tree<T>, Tree<T>, R> nodeFn) {

        return switch (this) {
            case Leaf<T> leaf -> leafFn.apply(leaf.getValue());
            case Node<T> node -> nodeFn.apply(node.getLeft(), node.getRight());
        };
    }
}

// Usage
Tree<Integer> tree = new Tree.Node<>(
    new Tree.Leaf<>(1),
    new Tree.Node<>(
        new Tree.Leaf<>(2),
        new Tree.Leaf<>(3)
    )
);

// Sum all values
int sum = sumTree(tree);

public int sumTree(Tree<Integer> tree) {
    return tree.match(
        value -> value,
        (left, right) -> sumTree(left) + sumTree(right)
    );
}
```

---

## Generics and Pattern Matching

### Pattern Matching for Generics (Java 21+)

```java
// Generic class with pattern matching
public sealed interface Optional<T> {
    record Some<T>(T value) implements Optional<T> {}
    record None<T>() implements Optional<T> {}
}

// Pattern matching with generics
public <T> String describe(Optional<T> opt) {
    return switch (opt) {
        case Optional.Some<T>(var value) -> "Present: " + value;
        case Optional.None<T>() -> "Empty";
    };
}

// Generic type patterns
public void process(Object obj) {
    if (obj instanceof List<?> list) {
        System.out.println("List size: " + list.size());
    }

    if (obj instanceof Map<?, ?> map) {
        System.out.println("Map size: " + map.size());
    }
}

// Record patterns with generics
public record Pair<A, B>(A first, B second) {}

public <A, B> void processPair(Pair<A, B> pair) {
    if (pair instanceof Pair(var first, var second)) {
        System.out.println("First: " + first);
        System.out.println("Second: " + second);
    }
}
```

---

## Advanced Patterns

### Self-Bounded Generics (Recursive Type Bound)

```java
// Self-bounded type - T must extend Comparable of itself
public abstract class Entity<T extends Entity<T>> implements Comparable<T> {
    protected Long id;

    @Override
    public int compareTo(T other) {
        return Long.compare(this.id, other.id);
    }

    // Can return exact subtype
    public abstract T withId(Long id);
}

public class User extends Entity<User> {
    private String name;

    @Override
    public User withId(Long id) {
        User user = new User();
        user.id = id;
        user.name = this.name;
        return user; // Returns User, not Entity
    }
}

// Fluent builder with self-bounded generics
public abstract class Builder<T, B extends Builder<T, B>> {

    @SuppressWarnings("unchecked")
    protected B self() {
        return (B) this;
    }

    public abstract T build();
}

public class UserBuilder extends Builder<User, UserBuilder> {
    private String name;
    private String email;

    public UserBuilder withName(String name) {
        this.name = name;
        return self(); // Returns UserBuilder, not Builder
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return self();
    }

    @Override
    public User build() {
        return new User(name, email);
    }
}

// Usage - fluent chaining works!
User user = new UserBuilder()
    .withName("John")
    .withEmail("john@example.com")
    .build();
```

### Type Tokens and Super Type Tokens

```java
// Type token for runtime type information
public class TypeReference<T> {
    private final Type type;

    protected TypeReference() {
        Type superclass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superclass).getActualTypeArguments()[0];
    }

    public Type getType() {
        return type;
    }
}

// Usage
TypeReference<List<String>> typeRef = new TypeReference<>() {};
Type type = typeRef.getType();
// type = java.util.List<java.lang.String>

// Practical use: JSON parsing
public class JsonParser {

    public <T> T parse(String json, TypeReference<T> typeRef) {
        // Use typeRef.getType() to parse with full generic info
        return null; // Actual implementation would use Jackson/Gson
    }
}

// Usage
List<User> users = jsonParser.parse(
    jsonString,
    new TypeReference<List<User>>() {}
);
```

### Phantom Types

```java
// Phantom types for compile-time state tracking
public interface State {}
public interface Validated extends State {}
public interface Unvalidated extends State {}

public class Form<S extends State> {
    private final Map<String, String> fields;

    private Form(Map<String, String> fields) {
        this.fields = fields;
    }

    public static Form<Unvalidated> create() {
        return new Form<>(new HashMap<>());
    }

    public Form<Unvalidated> withField(String key, String value) {
        fields.put(key, value);
        return this;
    }

    public Form<Validated> validate() {
        // Validation logic
        if (fields.isEmpty()) {
            throw new IllegalStateException("No fields to validate");
        }
        return new Form<>(this.fields);
    }

    // Only validated forms can be submitted
    public void submit(Form<Validated> form) {
        // Submit the form
    }
}

// Usage
Form<Unvalidated> form = Form.create()
    .withField("name", "John")
    .withField("email", "john@example.com");

// form.submit(form); // Compile error!

Form<Validated> validated = form.validate();
validated.submit(validated); // OK!
```

---

## Common Pitfalls

### Pitfall 1: Raw Types

```java
// ❌ BAD: Using raw types
List list = new ArrayList(); // Raw type - no type safety
list.add("string");
list.add(42);

String str = (String) list.get(1); // ClassCastException!

// ✅ GOOD: Always use generics
List<String> list = new ArrayList<>();
list.add("string");
// list.add(42); // Compile error
```

### Pitfall 2: Heap Pollution

```java
// ❌ Heap pollution with varargs
@SafeVarargs // Suppresses warning, but method is NOT safe!
public static <T> void dangerousMethod(T... elements) {
    Object[] array = elements;
    array[0] = "String"; // Heap pollution!
}

List<Integer> list = new ArrayList<>();
list.add(42);
dangerousMethod(list); // ClassCastException later!

// ✅ GOOD: Safe varargs usage
@SafeVarargs
public static <T> List<T> safeMethod(T... elements) {
    return Arrays.asList(elements); // Safe - no modification
}
```

### Pitfall 3: Generic Exception Types

```java
// ❌ Cannot catch generic exceptions
public <T extends Exception> void method() {
    try {
        // some code
    } catch (T e) { // Compile error!
        // handle
    }
}

// ❌ Cannot create generic exception
public class GenericException<T> extends Exception { // Compile error!
}

// ✅ Can throw generic exceptions
public <T extends Exception> void throwIt(T exception) throws T {
    throw exception;
}
```

---

## Best Practices

### ✅ DO

```java
// Use meaningful type parameter names
public class Repository<Entity, Id> {} // Better than <T, U>

// Use bounded wildcards for API flexibility
public void processNumbers(List<? extends Number> numbers) {}

// Apply PECS principle
public <T> void copy(
    List<? extends T> source,
    List<? super T> dest
) {}

// Use type inference (diamond operator)
Map<String, List<Integer>> map = new HashMap<>(); // Not HashMap<String, List<Integer>>()

// Use @SafeVarargs appropriately
@SafeVarargs
public static <T> List<T> listOf(T... elements) {
    return Arrays.asList(elements);
}
```

### ❌ DON'T

```java
// Don't use raw types
List list = new ArrayList(); // ❌

// Don't ignore generic type
List<?> list = getList();
// list.add(something); // ❌ Can't add to unknown type

// Don't create generic arrays
T[] array = new T[10]; // ❌ Compile error

// Don't use instanceof with generic types
if (obj instanceof List<String>) {} // ❌ Compile error
if (obj instanceof List<?>) {} // ✅ OK

// Don't suppress warnings unnecessarily
@SuppressWarnings("unchecked")
List<String> list = (List<String>) obj; // Only if absolutely necessary
```

### Naming Conventions

```
Single letter type parameters:
- T: Type
- E: Element (used in collections)
- K: Key
- V: Value
- N: Number
- S, U, V: 2nd, 3rd, 4th types

Multi-letter for clarity:
- Entity, Id, Result, Response, etc.
```

---

## Production Examples

### Generic Repository Pattern

```java
public interface CrudRepository<T, ID> {
    T save(T entity);
    Optional<T> findById(ID id);
    List<T> findAll();
    void deleteById(ID id);
    boolean existsById(ID id);
}

public abstract class AbstractRepository<T, ID> implements CrudRepository<T, ID> {
    protected final MongoTemplate mongoTemplate;
    protected final Class<T> entityClass;

    @SuppressWarnings("unchecked")
    protected AbstractRepository(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
        this.entityClass = (Class<T>) ((ParameterizedType) getClass()
            .getGenericSuperclass())
            .getActualTypeArguments()[0];
    }

    @Override
    public T save(T entity) {
        return mongoTemplate.save(entity);
    }

    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(mongoTemplate.findById(id, entityClass));
    }

    @Override
    public List<T> findAll() {
        return mongoTemplate.findAll(entityClass);
    }
}

// Usage
public class UserRepository extends AbstractRepository<User, String> {
    public UserRepository(MongoTemplate mongoTemplate) {
        super(mongoTemplate);
    }

    public Optional<User> findByEmail(String email) {
        Query query = Query.query(Criteria.where("email").is(email));
        return Optional.ofNullable(mongoTemplate.findOne(query, entityClass));
    }
}
```

### Generic Response Wrapper

```java
public record ApiResponse<T>(
    T data,
    String message,
    int statusCode,
    LocalDateTime timestamp
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(data, "Success", 200, LocalDateTime.now());
    }

    public static <T> ApiResponse<T> error(String message, int statusCode) {
        return new ApiResponse<>(null, message, statusCode, LocalDateTime.now());
    }

    public boolean isSuccess() {
        return statusCode >= 200 && statusCode < 300;
    }
}

// Usage in controller
@GetMapping("/users/{id}")
public ApiResponse<User> getUser(@PathVariable String id) {
    return userService.findById(id)
        .map(ApiResponse::success)
        .orElseGet(() -> ApiResponse.error("User not found", 404));
}
```

---

*This guide provides comprehensive patterns for mastering Java generics. Generics enable type-safe, reusable code - use them effectively to write better Java applications.*
