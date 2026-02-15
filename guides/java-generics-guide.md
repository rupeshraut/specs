# Effective Modern Java Generics

A comprehensive guide to mastering Java generics, type safety, advanced patterns, and best practices.

---

## Table of Contents

1. [Generics Fundamentals](#generics-fundamentals)
2. [Type Parameters and Naming](#type-parameters-and-naming)
3. [Generic Classes and Interfaces](#generic-classes-and-interfaces)
4. [Generic Methods](#generic-methods)
5. [Bounded Type Parameters](#bounded-type-parameters)
6. [Wildcards](#wildcards)
7. [PECS Principle](#pecs-principle)
8. [Type Erasure](#type-erasure)
9. [Generic Collections](#generic-collections)
10. [Advanced Patterns](#advanced-patterns)
11. [Recursive Type Bounds](#recursive-type-bounds)
12. [Type Inference](#type-inference)
13. [Common Pitfalls](#common-pitfalls)
14. [Best Practices](#best-practices)

---

## Generics Fundamentals

### Why Generics?

**Before Generics (Java 1.4):**

```java
// ❌ No type safety
List list = new ArrayList();
list.add("String");
list.add(42);
list.add(new Date());

String str = (String) list.get(0);  // Manual casting
String str2 = (String) list.get(1); // ClassCastException at runtime!
```

**With Generics (Java 5+):**

```java
// ✅ Type safety at compile time
List<String> list = new ArrayList<>();
list.add("String");
// list.add(42);  // Compile error!

String str = list.get(0);  // No casting needed
```

**Benefits:**
- **Type Safety**: Catch errors at compile time
- **No Casting**: Cleaner, more readable code
- **Reusability**: Write once, use with any type
- **Documentation**: Types are self-documenting

---

## Type Parameters and Naming

### Naming Conventions

```java
/**
 * Standard Type Parameter Names:
 * 
 * E - Element (used extensively by Java Collections Framework)
 * K - Key (Map keys)
 * V - Value (Map values)
 * N - Number
 * T - Type
 * S, U, V - 2nd, 3rd, 4th types
 */

// Examples
class Box<T> { }                           // Generic type
interface Map<K, V> { }                    // Key-Value pair
class Pair<S, T> { }                       // Two types
interface Comparable<T> { }                // Type to compare with
class ArrayList<E> implements List<E> { } // Element type
```

### Single vs Multiple Type Parameters

```java
// Single type parameter
public class Container<T> {
    private T value;
    
    public void set(T value) {
        this.value = value;
    }
    
    public T get() {
        return value;
    }
}

// Multiple type parameters
public class Pair<K, V> {
    private K key;
    private V value;
    
    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }
    
    public K getKey() { return key; }
    public V getValue() { return value; }
}

// Usage
Container<String> stringContainer = new Container<>();
stringContainer.set("Hello");
String value = stringContainer.get();

Pair<String, Integer> pair = new Pair<>("Age", 30);
String key = pair.getKey();
Integer age = pair.getValue();
```

---

## Generic Classes and Interfaces

### Generic Class

```java
public class Repository<T> {
    private final List<T> items = new ArrayList<>();
    
    public void add(T item) {
        items.add(item);
    }
    
    public T get(int index) {
        return items.get(index);
    }
    
    public List<T> getAll() {
        return new ArrayList<>(items);
    }
    
    public void remove(T item) {
        items.remove(item);
    }
    
    public int size() {
        return items.size();
    }
}

// Usage
Repository<User> userRepo = new Repository<>();
userRepo.add(new User("John"));
User user = userRepo.get(0);
```

### Generic Interface

```java
public interface Validator<T> {
    boolean validate(T item);
    String getErrorMessage();
}

// Implementation
public class EmailValidator implements Validator<String> {
    
    @Override
    public boolean validate(String email) {
        return email != null && email.contains("@");
    }
    
    @Override
    public String getErrorMessage() {
        return "Invalid email format";
    }
}

// Generic implementation
public class RangeValidator<T extends Comparable<T>> implements Validator<T> {
    private final T min;
    private final T max;
    
    public RangeValidator(T min, T max) {
        this.min = min;
        this.max = max;
    }
    
    @Override
    public boolean validate(T item) {
        return item.compareTo(min) >= 0 && item.compareTo(max) <= 0;
    }
    
    @Override
    public String getErrorMessage() {
        return "Value must be between " + min + " and " + max;
    }
}

// Usage
Validator<String> emailValidator = new EmailValidator();
boolean valid = emailValidator.validate("user@example.com");

Validator<Integer> ageValidator = new RangeValidator<>(18, 100);
boolean validAge = ageValidator.validate(25);
```

### Nested Generic Classes

```java
public class Outer<T> {
    private T value;
    
    public class Inner<U> {
        private U innerValue;
        
        public void set(T outerValue, U innerValue) {
            Outer.this.value = outerValue;
            this.innerValue = innerValue;
        }
        
        public Pair<T, U> getPair() {
            return new Pair<>(value, innerValue);
        }
    }
    
    public Inner<String> createStringInner() {
        return new Inner<>();
    }
}

// Usage
Outer<Integer> outer = new Outer<>();
Outer<Integer>.Inner<String> inner = outer.new Inner<>();
inner.set(42, "Hello");
```

---

## Generic Methods

### Basic Generic Methods

```java
public class Utils {
    
    // Generic method
    public static <T> T getFirst(List<T> list) {
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
    
    // Generic method with bounded type
    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) > 0 ? a : b;
    }
    
    // Swap elements
    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}

// Usage
List<String> strings = Arrays.asList("a", "b", "c");
String first = Utils.getFirst(strings);

String[] keys = {"name", "age"};
Object[] values = {"John", 30};
Map<String, Object> map = Utils.createMap(keys, values);

Integer maxValue = Utils.max(10, 20);
String maxString = Utils.max("apple", "banana");
```

### Generic Methods in Generic Classes

```java
public class Container<T> {
    private T value;
    
    // Regular method using class type parameter
    public void setValue(T value) {
        this.value = value;
    }
    
    // Generic method with its own type parameter
    public <U> U transform(Function<T, U> transformer) {
        return transformer.apply(value);
    }
    
    // Generic method with different type than class
    public <U> Container<U> map(Function<T, U> mapper) {
        Container<U> newContainer = new Container<>();
        newContainer.setValue(mapper.apply(value));
        return newContainer;
    }
}

// Usage
Container<String> stringContainer = new Container<>();
stringContainer.setValue("123");

// Transform String to Integer
Integer number = stringContainer.transform(Integer::parseInt);

// Map to new container type
Container<Integer> intContainer = stringContainer.map(Integer::parseInt);
```

---

## Bounded Type Parameters

### Upper Bounds (extends)

```java
// Upper bound with class
public class NumberContainer<T extends Number> {
    private T value;
    
    public NumberContainer(T value) {
        this.value = value;
    }
    
    public double doubleValue() {
        return value.doubleValue();  // Can call Number methods
    }
}

// Usage
NumberContainer<Integer> intContainer = new NumberContainer<>(42);
NumberContainer<Double> doubleContainer = new NumberContainer<>(3.14);
// NumberContainer<String> stringContainer = new NumberContainer<>("text"); // Compile error!

// Upper bound with interface
public class SortedList<T extends Comparable<T>> {
    private final List<T> items = new ArrayList<>();
    
    public void add(T item) {
        items.add(item);
        Collections.sort(items);  // Can sort because T is Comparable
    }
    
    public T getMin() {
        return items.isEmpty() ? null : items.get(0);
    }
    
    public T getMax() {
        return items.isEmpty() ? null : items.get(items.size() - 1);
    }
}

// Multiple bounds (class must be first, interfaces follow)
public class DataProcessor<T extends Number & Serializable> {
    public void process(T data) {
        // Can use Number methods
        double value = data.doubleValue();
        
        // Can serialize
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(data);
    }
}
```

### Recursive Type Bounds

```java
// Enum pattern
public class Enum<E extends Enum<E>> implements Comparable<E> {
    @Override
    public int compareTo(E other) {
        return this.ordinal() - other.ordinal();
    }
}

// Builder pattern
public class Builder<T extends Builder<T>> {
    
    @SuppressWarnings("unchecked")
    protected T self() {
        return (T) this;
    }
    
    public T withName(String name) {
        // Set name
        return self();
    }
    
    public T withAge(int age) {
        // Set age
        return self();
    }
}

public class EmployeeBuilder extends Builder<EmployeeBuilder> {
    public EmployeeBuilder withSalary(double salary) {
        // Set salary
        return self();
    }
}

// Usage - fluent API
EmployeeBuilder builder = new EmployeeBuilder()
    .withName("John")
    .withAge(30)
    .withSalary(50000.0);
```

---

## Wildcards

### Unbounded Wildcard (?)

```java
// Accepts any type
public static void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Can read as Object, cannot write
public static int countElements(List<?> list) {
    return list.size();  // OK
    // list.add("item");  // Compile error!
}

// Usage
List<String> strings = Arrays.asList("a", "b", "c");
List<Integer> integers = Arrays.asList(1, 2, 3);
printList(strings);
printList(integers);
```

### Upper Bounded Wildcard (? extends T)

```java
// Read-only, accepts T or any subtype of T
public static double sumNumbers(List<? extends Number> numbers) {
    double sum = 0;
    for (Number num : numbers) {
        sum += num.doubleValue();  // Can read as Number
    }
    // numbers.add(42);  // Compile error! Cannot write
    return sum;
}

// Usage
List<Integer> integers = Arrays.asList(1, 2, 3);
List<Double> doubles = Arrays.asList(1.1, 2.2, 3.3);
List<Number> numbers = Arrays.asList(1, 2.5, 3L);

double sum1 = sumNumbers(integers);  // OK
double sum2 = sumNumbers(doubles);   // OK
double sum3 = sumNumbers(numbers);   // OK

// Copy from source (read)
public static <T> void copy(List<? extends T> source, List<T> dest) {
    for (T item : source) {
        dest.add(item);  // Read from source, write to dest
    }
}
```

### Lower Bounded Wildcard (? super T)

```java
// Write-only, accepts T or any supertype of T
public static void addNumbers(List<? super Integer> list) {
    list.add(1);      // Can write Integer
    list.add(2);
    list.add(3);
    
    // Object obj = list.get(0);  // Can only read as Object
    // Integer num = list.get(0); // Compile error!
}

// Usage
List<Integer> integers = new ArrayList<>();
List<Number> numbers = new ArrayList<>();
List<Object> objects = new ArrayList<>();

addNumbers(integers);  // OK
addNumbers(numbers);   // OK
addNumbers(objects);   // OK

// Comparator example
public static <T> void sort(List<T> list, Comparator<? super T> comparator) {
    list.sort(comparator);
}

// Can use comparator for T or any supertype
List<Integer> ints = Arrays.asList(3, 1, 2);
Comparator<Number> numberComparator = Comparator.comparing(Number::doubleValue);
sort(ints, numberComparator);  // OK - Comparator<Number> works for List<Integer>
```

---

## PECS Principle

**PECS: Producer Extends, Consumer Super**

### Producer Extends

```java
// Producer: You want to READ from the structure (it produces items)
// Use <? extends T>

public class Stack<E> {
    private final List<E> elements = new ArrayList<>();
    
    // Producer: Reading from source (source produces items)
    public void pushAll(Iterable<? extends E> src) {
        for (E e : src) {
            elements.add(e);
        }
    }
}

// Usage
Stack<Number> numberStack = new Stack<>();
List<Integer> integers = Arrays.asList(1, 2, 3);
List<Double> doubles = Arrays.asList(1.1, 2.2);

numberStack.pushAll(integers);  // Integer extends Number
numberStack.pushAll(doubles);   // Double extends Number
```

### Consumer Super

```java
// Consumer: You want to WRITE to the structure (it consumes items)
// Use <? super T>

public class Stack<E> {
    private final List<E> elements = new ArrayList<>();
    
    // Consumer: Writing to destination (destination consumes items)
    public void popAll(Collection<? super E> dst) {
        while (!elements.isEmpty()) {
            dst.add(elements.remove(elements.size() - 1));
        }
    }
}

// Usage
Stack<Integer> intStack = new Stack<>();
intStack.pushAll(Arrays.asList(1, 2, 3));

Collection<Integer> integers = new ArrayList<>();
Collection<Number> numbers = new ArrayList<>();
Collection<Object> objects = new ArrayList<>();

intStack.popAll(integers);  // OK
intStack.popAll(numbers);   // OK - Number is super of Integer
intStack.popAll(objects);   // OK - Object is super of Integer
```

### PECS in Collections API

```java
// Collections.copy uses PECS
public static <T> void copy(
    List<? super T> dest,     // Consumer - writes to dest
    List<? extends T> src     // Producer - reads from src
) {
    for (T item : src) {
        dest.add(item);
    }
}

// Usage
List<Integer> integers = Arrays.asList(1, 2, 3);
List<Number> numbers = new ArrayList<>();

Collections.copy(numbers, integers);  // OK - reads Integer, writes Number

// Collections.max uses producer
public static <T extends Object & Comparable<? super T>> T max(
    Collection<? extends T> coll  // Producer - reads from collection
) {
    // Implementation
}
```

---

## Type Erasure

### Understanding Type Erasure

```java
// Source code
public class Box<T> {
    private T value;
    
    public void set(T value) {
        this.value = value;
    }
    
    public T get() {
        return value;
    }
}

// After type erasure (what JVM sees)
public class Box {
    private Object value;  // T becomes Object
    
    public void set(Object value) {
        this.value = value;
    }
    
    public Object get() {
        return value;
    }
}

// With bounds: <T extends Number>
// After erasure: T becomes Number (not Object)
```

### Implications of Type Erasure

```java
// ❌ Cannot create instance of type parameter
public class Container<T> {
    public T create() {
        // return new T();  // Compile error!
    }
}

// ✅ Solution: Pass class object
public class Container<T> {
    private final Class<T> type;
    
    public Container(Class<T> type) {
        this.type = type;
    }
    
    public T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}

// ❌ Cannot create generic array
public class GenericArray<T> {
    // private T[] array = new T[10];  // Compile error!
}

// ✅ Solution: Use ArrayList or Object array with cast
public class GenericArray<T> {
    private final List<T> list = new ArrayList<>();
    
    // Or with Object array
    @SuppressWarnings("unchecked")
    private final T[] array = (T[]) new Object[10];
}

// ❌ Cannot use instanceof with parameterized types
if (obj instanceof List<String>) { }  // Compile error!

// ✅ Can use with raw type or wildcard
if (obj instanceof List<?>) { }  // OK
if (obj instanceof List) { }     // OK (raw type)

// ❌ Cannot catch or throw generic exception
try {
    // Something
} catch (T e) { }  // Compile error!

// ❌ Cannot have overloaded methods that erase to same signature
public void process(List<String> list) { }
public void process(List<Integer> list) { }  // Compile error! Same erasure
```

---

## Generic Collections

### Common Generic Collections

```java
// List
List<String> arrayList = new ArrayList<>();
List<String> linkedList = new LinkedList<>();
List<String> immutableList = List.of("a", "b", "c");  // Java 9+

// Set
Set<String> hashSet = new HashSet<>();
Set<String> linkedHashSet = new LinkedHashSet<>();
Set<String> treeSet = new TreeSet<>();
Set<String> immutableSet = Set.of("a", "b", "c");

// Map
Map<String, Integer> hashMap = new HashMap<>();
Map<String, Integer> linkedHashMap = new LinkedHashMap<>();
Map<String, Integer> treeMap = new TreeMap<>();
Map<String, Integer> immutableMap = Map.of("a", 1, "b", 2);

// Queue
Queue<String> linkedListQueue = new LinkedList<>();
Queue<String> priorityQueue = new PriorityQueue<>();
Deque<String> arrayDeque = new ArrayDeque<>();

// Concurrent collections
Map<String, Integer> concurrentHashMap = new ConcurrentHashMap<>();
Queue<String> concurrentLinkedQueue = new ConcurrentLinkedQueue<>();
```

### Type-Safe Collections

```java
public class UserRepository {
    private final Map<String, User> users = new HashMap<>();
    private final List<User> userList = new ArrayList<>();
    
    public void addUser(User user) {
        users.put(user.getId(), user);
        userList.add(user);
    }
    
    public Optional<User> findById(String id) {
        return Optional.ofNullable(users.get(id));
    }
    
    public List<User> findAll() {
        return new ArrayList<>(userList);  // Defensive copy
    }
    
    public List<User> findByPredicate(Predicate<User> predicate) {
        return userList.stream()
            .filter(predicate)
            .collect(Collectors.toList());
    }
}

// Usage
UserRepository repo = new UserRepository();
repo.addUser(new User("1", "John"));

Optional<User> user = repo.findById("1");
List<User> allUsers = repo.findAll();
List<User> adults = repo.findByPredicate(u -> u.getAge() >= 18);
```

---

## Advanced Patterns

### Generic Factory Pattern

```java
public interface Factory<T> {
    T create();
}

public class UserFactory implements Factory<User> {
    @Override
    public User create() {
        return new User();
    }
}

// Generic factory with parameters
public interface ParameterizedFactory<T, P> {
    T create(P param);
}

public class UserFromIdFactory implements ParameterizedFactory<User, String> {
    @Override
    public User create(String id) {
        return new User(id);
    }
}

// Factory registry
public class FactoryRegistry {
    private final Map<Class<?>, Factory<?>> factories = new HashMap<>();
    
    public <T> void register(Class<T> type, Factory<T> factory) {
        factories.put(type, factory);
    }
    
    @SuppressWarnings("unchecked")
    public <T> T create(Class<T> type) {
        Factory<T> factory = (Factory<T>) factories.get(type);
        if (factory == null) {
            throw new IllegalArgumentException("No factory for " + type);
        }
        return factory.create();
    }
}

// Usage
FactoryRegistry registry = new FactoryRegistry();
registry.register(User.class, new UserFactory());

User user = registry.create(User.class);
```

### Generic Builder Pattern

```java
public abstract class Builder<T extends Builder<T, R>, R> {
    
    @SuppressWarnings("unchecked")
    protected T self() {
        return (T) this;
    }
    
    public abstract R build();
}

public class User {
    private final String name;
    private final String email;
    private final int age;
    
    private User(UserBuilder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
    }
    
    public static class UserBuilder extends Builder<UserBuilder, User> {
        private String name;
        private String email;
        private int age;
        
        public UserBuilder withName(String name) {
            this.name = name;
            return self();
        }
        
        public UserBuilder withEmail(String email) {
            this.email = email;
            return self();
        }
        
        public UserBuilder withAge(int age) {
            this.age = age;
            return self();
        }
        
        @Override
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.UserBuilder()
    .withName("John")
    .withEmail("john@example.com")
    .withAge(30)
    .build();
```

### Generic Strategy Pattern

```java
public interface Strategy<T, R> {
    R execute(T input);
}

public class Context<T, R> {
    private Strategy<T, R> strategy;
    
    public void setStrategy(Strategy<T, R> strategy) {
        this.strategy = strategy;
    }
    
    public R executeStrategy(T input) {
        if (strategy == null) {
            throw new IllegalStateException("Strategy not set");
        }
        return strategy.execute(input);
    }
}

// Concrete strategies
public class UpperCaseStrategy implements Strategy<String, String> {
    @Override
    public String execute(String input) {
        return input.toUpperCase();
    }
}

public class LengthStrategy implements Strategy<String, Integer> {
    @Override
    public Integer execute(String input) {
        return input.length();
    }
}

// Usage
Context<String, String> stringContext = new Context<>();
stringContext.setStrategy(new UpperCaseStrategy());
String result = stringContext.executeStrategy("hello");  // "HELLO"

Context<String, Integer> intContext = new Context<>();
intContext.setStrategy(new LengthStrategy());
Integer length = intContext.executeStrategy("hello");  // 5
```

### Generic Visitor Pattern

```java
public interface Visitor<T, R> {
    R visit(T element);
}

public interface Visitable<T> {
    <R> R accept(Visitor<T, R> visitor);
}

public class Document implements Visitable<Document> {
    private final String content;
    
    public Document(String content) {
        this.content = content;
    }
    
    public String getContent() {
        return content;
    }
    
    @Override
    public <R> R accept(Visitor<Document, R> visitor) {
        return visitor.visit(this);
    }
}

// Concrete visitors
public class WordCountVisitor implements Visitor<Document, Integer> {
    @Override
    public Integer visit(Document document) {
        return document.getContent().split("\\s+").length;
    }
}

public class UpperCaseVisitor implements Visitor<Document, String> {
    @Override
    public String visit(Document document) {
        return document.getContent().toUpperCase();
    }
}

// Usage
Document doc = new Document("Hello World");

Integer wordCount = doc.accept(new WordCountVisitor());
String upperCase = doc.accept(new UpperCaseVisitor());
```

---

## Recursive Type Bounds

### Self-Referencing Types

```java
// Comparable pattern
public interface Comparable<T extends Comparable<T>> {
    int compareTo(T other);
}

// Implementation
public class Person implements Comparable<Person> {
    private final String name;
    private final int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}

// Generic max with recursive bound
public static <T extends Comparable<T>> T max(List<T> list) {
    if (list.isEmpty()) {
        throw new IllegalArgumentException("Empty list");
    }
    
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}
```

### Builder with Inheritance

```java
public abstract class Animal<T extends Animal<T>> {
    private String name;
    
    @SuppressWarnings("unchecked")
    protected T self() {
        return (T) this;
    }
    
    public T setName(String name) {
        this.name = name;
        return self();
    }
    
    public String getName() {
        return name;
    }
}

public class Dog extends Animal<Dog> {
    private String breed;
    
    public Dog setBreed(String breed) {
        this.breed = breed;
        return self();
    }
    
    public String getBreed() {
        return breed;
    }
}

// Usage - fluent API works correctly
Dog dog = new Dog()
    .setName("Buddy")      // Returns Dog, not Animal
    .setBreed("Labrador"); // Can chain Dog-specific methods
```

---

## Type Inference

### Diamond Operator (Java 7+)

```java
// Before Java 7
List<String> list = new ArrayList<String>();
Map<String, List<Integer>> map = new HashMap<String, List<Integer>>();

// Java 7+ with diamond operator
List<String> list = new ArrayList<>();
Map<String, List<Integer>> map = new HashMap<>();

// Works with nested generics
List<Map<String, List<Integer>>> complex = new ArrayList<>();
```

### Type Inference in Methods

```java
// Type inference from arguments
public static <T> List<T> asList(T... elements) {
    return Arrays.asList(elements);
}

// Usage - type inferred from arguments
List<String> strings = asList("a", "b", "c");  // T inferred as String
List<Integer> integers = asList(1, 2, 3);      // T inferred as Integer

// Explicit type witness (when needed)
List<String> empty = Collections.<String>emptyList();

// Target-type inference (Java 8+)
List<String> list = new ArrayList<>();
list.add("hello");

// Stream API with type inference
List<String> strings = Arrays.asList("a", "b", "c");
List<Integer> lengths = strings.stream()
    .map(String::length)  // Type inferred
    .collect(Collectors.toList());
```

### Var Keyword (Java 10+)

```java
// Local variable type inference
var list = new ArrayList<String>();  // ArrayList<String>
var map = new HashMap<String, Integer>();  // HashMap<String, Integer>
var stream = list.stream();  // Stream<String>

// With generic methods
var result = Arrays.asList("a", "b", "c");  // List<String>

// ❌ Cannot use with null
// var x = null;  // Compile error!

// ❌ Cannot use without initializer
// var y;  // Compile error!

// ✅ Use when type is obvious
var userList = new ArrayList<User>();
var userMap = new HashMap<String, User>();

// ❌ Don't overuse - prefer explicit types when clarity matters
var x = getResult();  // What type is x? Not clear!
```

---

## Common Pitfalls

### Pitfall 1: Raw Types

```java
// ❌ BAD: Using raw types
List list = new ArrayList();
list.add("String");
list.add(123);
String str = (String) list.get(1);  // ClassCastException!

// ✅ GOOD: Use generics
List<String> list = new ArrayList<>();
list.add("String");
// list.add(123);  // Compile error!
```

### Pitfall 2: Generic Array Creation

```java
// ❌ BAD: Cannot create generic array
public class Container<T> {
    // private T[] array = new T[10];  // Compile error!
}

// ✅ GOOD: Use List or cast
public class Container<T> {
    private final List<T> list = new ArrayList<>();
    
    // Or
    @SuppressWarnings("unchecked")
    private final T[] array = (T[]) new Object[10];
}
```

### Pitfall 3: Type Erasure Confusion

```java
// ❌ BAD: Thinking generics work at runtime
public <T> void process(List<T> list) {
    // if (list instanceof List<String>) { }  // Compile error!
}

// ✅ GOOD: Use class parameter for runtime check
public <T> void process(List<T> list, Class<T> type) {
    if (type == String.class) {
        // Process as strings
    }
}
```

### Pitfall 4: Wildcard Confusion

```java
// ❌ BAD: Cannot write to producer
public void bad(List<? extends Number> list) {
    // list.add(42);  // Compile error!
}

// ✅ GOOD: Remember PECS
public void addNumbers(List<? super Integer> list) {
    list.add(42);  // OK - Consumer
}

public double sum(List<? extends Number> list) {
    return list.stream()
        .mapToDouble(Number::doubleValue)
        .sum();  // OK - Producer
}
```

### Pitfall 5: Missing Bounds

```java
// ❌ BAD: No bounds when needed
public <T> T findMax(List<T> list) {
    // Cannot compare T objects!
    // return Collections.max(list);  // Compile error!
}

// ✅ GOOD: Add appropriate bound
public <T extends Comparable<T>> T findMax(List<T> list) {
    return Collections.max(list);  // OK
}
```

---

## Best Practices

### ✅ DO

1. **Use generics instead of raw types**
   ```java
   List<String> list = new ArrayList<>();  // ✅
   List list = new ArrayList();             // ❌
   ```

2. **Favor immutable generic collections**
   ```java
   List<String> immutable = List.of("a", "b", "c");
   Set<Integer> immutableSet = Set.of(1, 2, 3);
   ```

3. **Use bounded wildcards for API flexibility**
   ```java
   public void process(List<? extends Number> numbers) { }
   ```

4. **Apply PECS principle**
   ```java
   public <T> void copy(List<? super T> dest, List<? extends T> src) { }
   ```

5. **Use diamond operator**
   ```java
   Map<String, List<Integer>> map = new HashMap<>();
   ```

6. **Prefer generic methods**
   ```java
   public static <T> List<T> asList(T... items) {
       return Arrays.asList(items);
   }
   ```

7. **Document type parameters**
   ```java
   /**
    * @param <T> the type of elements in this list
    */
   public interface List<T> { }
   ```

8. **Use meaningful type parameter names**
   ```java
   public interface Map<K, V> { }  // ✅ Clear
   public interface Map<T, U> { }  // ❌ Less clear
   ```

9. **Return most specific type**
   ```java
   public ArrayList<String> getList() {  // ✅ Specific
       return new ArrayList<>();
   }
   ```

10. **Use @SafeVarargs for varargs**
    ```java
    @SafeVarargs
    public static <T> List<T> asList(T... items) {
        return Arrays.asList(items);
    }
    ```

### ❌ DON'T

1. **Don't use raw types**
2. **Don't ignore compiler warnings**
3. **Don't create generic exceptions**
4. **Don't use generics with primitives** (use wrappers)
5. **Don't use instanceof with parameterized types**
6. **Don't create arrays of generic types**
7. **Don't cast unnecessarily**
8. **Don't overuse wildcards** (keep it simple)
9. **Don't forget type erasure limitations**
10. **Don't mix raw and generic types**

---

## Conclusion

**Key Takeaways:**

1. **Type Safety**: Generics provide compile-time type checking
2. **PECS**: Producer Extends, Consumer Super
3. **Bounded Types**: Use `extends` for upper bounds
4. **Wildcards**: Use `?`, `? extends T`, `? super T` appropriately
5. **Type Erasure**: Understand runtime limitations
6. **Collections**: Prefer generic collections over raw types
7. **Patterns**: Use generics to create reusable, type-safe components

**Remember**: Generics are about writing flexible, reusable, and type-safe code. Start with basics, understand PECS, and practice with real-world examples!
