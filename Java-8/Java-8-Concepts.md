# Java 8 – New/Enhanced Features

## 1. Lambda Expressions

### Syntax
```java
// Basic lambda
() -> System.out.println("Hello");

// With parameter
x -> x * x;

// With multiple parameters and typed declaration
(int a, int b) -> a + b;

// With explicit type
(String s) -> s.length();
```

### Lambda with Functional Interfaces
```java
Runnable r = () -> System.out.println("Running");
Comparator<String> cmp = (a, b) -> a.compareTo(b);
Function<Integer, String> intToStr = x -> "Num: " + x;
Predicate<String> isEmpty = s -> s.isEmpty();
Consumer<String> printer = s -> System.out.println(s);
Supplier<Double> random = () -> Math.random();
```

### Variable Capture
```java
// Lambda can capture effectively final variables
int factor = 2; // must be effectively final
UnaryOperator<Integer> multiply = x -> x * factor;
// factor = 3; // Compile error — cannot modify
```

### Method References
```java
// Four types:
// 1. Static method: ClassName::staticMethod
Function<String, Integer> len = String::length;

// 2. Instance method (specific): instance::method
String str = "hello";
Supplier<Integer> len2 = str::length;

// 3. Instance method (arbitrary): ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;

// 4. Constructor: ClassName::new
Supplier<List> listSupplier = ArrayList::new;
```

---

## 2. Functional Interfaces

### Core Interfaces (java.util.function)
```java
// Predicate<T> — returns boolean
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> isEmpty = String::isEmpty;
Predicate<String> combined = isLong.and(isEmpty.negate());

// Function<T, R> — transforms T to R
Function<String, Integer> length = String::length;
Function<String, String> upper = String::toUpperCase;
Function<String, String> andThen = length.and(String::valueOf);

// Supplier<T> — provides T (no input)
Supplier<Double> random = Math::random;
Supplier<List<String>> emptyList = ArrayList::new;

// Consumer<T> — consumes T (no return)
Consumer<String> printer = System.out::println;
Consumer<String> doublePrint = printer.andThen(s -> System.out.println(s.toUpperCase()));

// BiFunction<T, U, R> — two inputs, one output
BiFunction<Integer, Integer, String> addStr = (a, b) -> String.valueOf(a + b);

// UnaryOperator<T> — same input/output type
UnaryOperator<String> trim = String::trim;
```

### Primitive Specializations
```java
// Avoid boxing overhead
IntPredicate isEven = x -> x % 2 == 0;
IntUnaryOperator square = x -> x * x;
IntSupplier randomInt = () -> new Random().nextInt();
IntConsumer print = System.out::println;

// Other variants: Long*, Double*, Bi* variants
```

### Custom Functional Interface
```java
@FunctionalInterface
interface MathOperation {
    int operate(int a, int b);
    
    // Default methods allowed
    default void log(String msg) {
        System.out.println(msg);
    }
}

MathOperation add = (a, b) -> a + b;
```

---

## 3. Stream API

### Creating Streams
```java
List<String> names = Arrays.asList("John", "Jane", "Alice", "Bob");

// From Collection
Stream<String> s1 = names.stream();

// From values
Stream<String> s2 = Stream.of("a", "b", "c");

// From array
String[] arr = {"x", "y", "z"};
Stream<String> s3 = Arrays.stream(arr);

// Empty
Stream<String> empty = Stream.empty();

// Generate/iterate
Stream<Double> randoms = Stream.generate(Math::random);
Stream<Integer> odds = Stream.iterate(1, n -> n + 2);
Stream<Integer> limited = Stream.iterate(0, n -> n + 1).limit(10);

// Parallel
Stream<String> parallel = names.parallelStream();
```

### Intermediate Operations (Lazy)
```java
// filter — keep matching elements
names.stream().filter(s -> s.length() > 3);

// map — transform each element
names.stream().map(String::toUpperCase);

// flatMap — flatten nested structures
List<List<String>> nested = Arrays.asList(
    Arrays.asList("a", "b"),
    Arrays.asList("c", "d")
);
List<String> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList()); // [a, b, c, d]

// mapToInt/mapToLong/mapToDouble — primitive specialization
List<Integer> nums = Arrays.asList(1, 2, 3);
int sum = nums.stream().mapToInt(Integer::intValue).sum();

// distinct
names.stream().distinct();

// sorted
names.stream().sorted();
names.stream().sorted(Comparator.reverseOrder());

// limit / skip
Stream.iterate(1, n -> n + 1).limit(5); // [1,2,3,4,5]
Stream.iterate(1, n -> n + 1).skip(2).limit(3); // [3,4,5]

// peek — debugging (does NOT terminate)
names.stream().peek(System.out::println).collect(Collectors.toList());

// sequential — convert parallel back to sequential
names.parallelStream().sequential();
```

### Terminal Operations (Eager)
```java
// collect — accumulate into result
List<String> result = names.stream().collect(Collectors.toList());
Set<String> resultSet = names.stream().collect(Collectors.toSet());

// toList (Java 16+)
List<String> result2 = names.stream().toList();

// forEach
names.stream().forEach(System.out::println);

// forEachOrdered (parallel-safe order)
names.parallelStream().forEachOrdered(System.out::println);

// reduce
int sum = nums.stream().reduce(0, Integer::sum);
Optional<Integer> product = nums.stream().reduce((a, b) -> a * b);

// count
long count = names.stream().filter(s -> s.length() > 3).count();

// min / max
Optional<String> min = names.stream().min(Comparator.comparingInt(String::length));

// anyMatch / allMatch / noneMatch
boolean anyLong = names.stream().anyMatch(s -> s.length() > 5);
boolean allLong = names.stream().allMatch(s -> s.length() > 1);

// findFirst / findAny
Optional<String> first = names.stream().filter(s -> s.startsWith("J")).findFirst();
Optional<String> any = names.parallelStream().filter(s -> s.startsWith("J")).findAny();

// toArray
String[] array = names.stream().toArray(String[]::new);
```

### Collectors (java.util.stream.Collectors)
```java
// Grouping
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));

// Grouping with downstream collector
Map<Integer, Set<String>> byLengthDistinct = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.toSet()));

// Partitioning
Map<Boolean, List<String>> partitioned = names.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 3));

// Joining
String joined = names.stream().collect(Collectors.joining(", ", "[", "]"));

// Summarizing
IntSummaryStatistics stats = nums.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));
// stats.getSum(), stats.getAverage(), stats.getMin(), stats.getMax()

// Counting
Long count = names.stream().collect(Collectors.counting());

// Mapping downstream
Set<String> upperNames = names.stream()
    .collect(Collectors.mapping(String::toUpperCase, Collectors.toSet()));

// toMap
Map<String, Integer> nameToLength = names.stream()
    .collect(Collectors.toMap(s -> s, s -> s.length()));

// toMap with merge function (handle duplicates)
Map<String, Integer> nameToLengthSafe = names.stream()
    .collect(Collectors.toMap(s -> s, s -> s.length(), (existing, replacement) -> existing));

// groupingBy with counting
Map<String, Long> wordCounts = words.stream()
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));

// flatMapping (Java 9+)
List<List<String>> nested = ...;
List<String> flat = nested.stream()
    .collect(Collectors.flatMapping(List::stream, Collectors.toList()));
```

### Parallel Streams
```java
// Automatic parallelization
int sum = nums.parallelStream()
    .filter(n -> n > 10)
    .mapToInt(Integer::intValue)
    .sum();

// When to use: large data, CPU-intensive operations
// When NOT to use: small data, I/O operations, stateful operations, order-sensitive

// Custom fork-join pool
ForkJoinPool customPool = new ForkJoinPool(4);
int result = customPool.submit(() ->
    nums.parallelStream().mapToInt(Integer::intValue).sum()
).get();
```

---

## 4. Default & Static Methods in Interfaces

```java
interface Animal {
    // Abstract method
    void makeSound();
    
    // Default method — has implementation
    default void sleep() {
        System.out.println("Sleeping...");
    }
    
    // Static method — belongs to interface
    static boolean isMammal(Class<?> clazz) {
        return Mammal.class.isAssignableFrom(clause);
    }
}

class Dog implements Animal {
    @Override
    public void makeSound() {
        System.out.println("Woof!");
    }
    // Inherits sleep() default method
}

// Using default method from multiple interfaces (conflict resolution)
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() {
        A.super.hello(); // Explicitly choose A's default
    }
}
```

---

## 5. Optional

```java
// Creating Optionals
Optional<String> opt1 = Optional.of("value");       // non-null value
Optional<String> opt2 = Optional.ofNullable(null);   // null-safe
Optional<String> opt3 = Optional.empty();            // empty

// Checking
optional.isPresent();       // true if has value
optional.isEmpty();          // true if empty (Java 11+)
optional.isPresent();

// Extracting values
String value = optional.get();                        // throws if empty
String safe = optional.orElse("default");             // default value
String computed = optional.orElseGet(() -> compute()); // lazy default
// Throws exception if empty
String forced = optional.orElseThrow(IllegalStateException::new);

// Transform
Optional<String> upper = optional.map(String::toUpperCase());
Optional<Integer> length = optional.flatMap(s -> Optional.of(s.length()));

// Filter
Optional<String> filtered = optional.filter(s -> s.length() > 3);

// ifPresent / ifPresentOrElse (Java 9+)
optional.ifPresent(System.out::println);
optional.ifPresentOrElse(
    System.out::println,
    () -> System.out.println("Empty")
);

// ifPresent with stream (Java 9+)
optional.stream().forEach(System.out::println);

// toStream (Java 9+)
optional.stream().count(); // 0 or 1
```

### Anti-patterns to avoid
```java
// ❌ Don't use Optional as field, parameter, or return type in collections
// ❌ Don't use Optional.get() without check
// ❌ Don't use Optional for null checks in method parameters
// ✅ Use Optional for return types that may have no result
```

---

## 6. New Date/Time API (java.time)

### Core Classes
```java
// Instant — point on timeline (UTC)
Instant now = Instant.now(); // 2026-09-18T11:10:48Z
Instant specific = Instant.parse("2026-01-01T00:00:00Z");

// LocalDate — date only (no time, no zone)
LocalDate today = LocalDate.now();
LocalDate specificDate = LocalDate.of(2026, 9, 18);
LocalDate parsed = LocalDate.parse("2026-09-18");

// LocalTime — time only (no date, no zone)
LocalTime now = LocalTime.now();
LocalTime specificTime = LocalTime.of(14, 30, 0);

// LocalDateTime — date + time (no zone)
LocalDateTime now = LocalDateTime.now();
LocalDateTime combined = LocalDate.of(2026, 1, 1).atTime(0, 0);

// ZonedDateTime — full date/time with timezone
ZonedDateTime zoned = ZonedDateTime.now(ZoneId.of("America/New_York"));
ZonedDateTime utc = ZonedDateTime.now(ZoneOffset.UTC);

// Duration — time-based (seconds, nanoseconds)
Duration dur = Duration.between(start, end);
Duration.ofHours(2); Duration.ofMinutes(30);

// Period — date-based (years, months, days)
Period p = Period.between(LocalDate.of(2020, 1, 1), LocalDate.now());
Period.ofYears(1); Period.ofMonths(3); Period.ofDays(7);
```

### Operations
```java
LocalDate today = LocalDate.of(2026, 9, 18);

// Manipulation (immutable, returns new instance)
LocalDate tomorrow = today.plusDays(1);
LocalDate lastWeek = today.minusWeeks(1);
LocalDate nextMonth = today.plusMonths(1);
LocalDate firstOfMonth = today.withDayOfMonth(1);
LocalDate changed = today.withMonth(12).withYear(2027);

// Querying
DayOfWeek day = today.getDayOfWeek();
int month = today.getMonthValue();
boolean leap = today.isLeapYear();

// Comparing
boolean isAfter = today.isAfter(LocalDate.of(2026, 1, 1));
boolean isBefore = today.isBefore(LocalDate.of(2027, 1, 1));
int comparison = today.compareTo(LocalDate.of(2026, 6, 1));

// Formatting
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String formatted = today.format(formatter);
LocalDate parsed = LocalDate.parse("18/09/2026", formatter);

// Time zone conversion
ZonedDateTime nyTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
ZonedDateTime lonTime = nyTime.withZoneSameInstant(ZoneId.of("Europe/London"));

// Epoch
long epochDay = today.toEpochDay();
Instant instant = today.atStartOfDay(ZoneOffset.UTC).toInstant();
```

### Common Patterns
```java
// Calculate age
LocalDate birth = LocalDate.of(1990, 5, 15);
LocalDate now = LocalDate.now();
Period age = Period.between(birth, now);
int years = age.getYears();

// Find next Friday
LocalDate nextFriday = LocalDate.now().with(TemporalAdjusters.next(DayOfWeek.FRIDAY));

// Working days between dates
long workingDays = ChronoUnit.DAYS.between(start, end) 
    - ChronoUnit.DAYS.between(start, end) / 7 * 2; // rough

// Timestamp to LocalDateTime
Instant instant = Instant.now();
LocalDateTime ldt = instant.atZone(ZoneId.systemDefault()).toLocalDateTime();
```

---

## 7. CompletableFuture

```java
// Basic creation
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
    // Async task
    sleep(1000);
    return "Result";
});

CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
    // Async task with no return
    sleep(1000);
});

// Chaining transformations
CompletableFuture<String> chain = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")           // transform result
    .thenApply(String::toUpperCase);

// Consume result (no return)
CompletableFuture<Void> consume = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenAccept(System.out::println);

// Execute after completion (no access to result)
CompletableFuture<Void> execute = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenRun(() -> System.out.println("Done"));

// Combine two futures
CompletableFuture<String> combined = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenCombine(CompletableFuture.supplyAsync(() -> "World"), 
        (a, b) -> a + " " + b);

// Wait for all
CompletableFuture<Void> allOf = CompletableFuture.allOf(cf1, cf2, cf3);
allOf.join(); // Wait for all

// Wait for any
CompletableFuture<Object> anyOf = CompletableFuture.anyOf(cf1, cf2, cf3);
Object result = anyOf.join();

// Exception handling
CompletableFuture<String> handled = CompletableFuture
    .supplyAsync(() -> { throw new RuntimeException("Error"); })
    .exceptionally(ex -> "Recovered from: " + ex.getMessage());

CompletableFuture<String> handled2 = CompletableFuture
    .supplyAsync(() -> "Success")
    .handle((result1, exception) -> {
        if (exception != null) return "Error";
        return result1;
    });

// Async variants (run callback in separate thread)
cf.thenApplyAsync(s -> s.toUpperCase());
cf.thenAcceptAsync(System.out::println);
cf.thenRunAsync(() -> System.out.println("Done"));

// Compose (flatMap-like)
CompletableFuture<String> composed = CompletableFuture
    .supplyAsync(() -> 42)
    .thenCompose(num -> CompletableFuture.supplyAsync(() -> "Result: " + num));

// Cancellation
cf.cancel(true); // mayInterruptIfRunning
cf.isCancelled();
```

---

## 8. Base64 API

```java
import java.util.Base64;

// Basic encoding
String encoded = Base64.getEncoder().encodeToString("Hello World".getBytes());
byte[] decoded = Base64.getDecoder().decode(encoded);

// URL-safe encoding
String urlEncoded = Base64.getUrlEncoder().encodeToString(data);
byte[] urlDecoded = Base64.getUrlDecoder().decode(urlEncoded);

// Mime encoding (with line breaks)
String mimeEncoded = Base64.getMimeEncoder().encodeToString(data);

// With prefix/suffix
String custom = Base64.getEncoder().withoutPadding().encodeToString(data);
```

---

## 9. Repeating Annotations

```java
// Step 1: Define the repeatable annotation
@Repeatable(MyAnnotations.class)
@interface MyAnnotation {
    String value();
}

// Step 2: Define the container annotation
@interface MyAnnotations {
    MyAnnotation[] value();
}

// Step 3: Use it repeatedly
@MyAnnotation("First")
@MyAnnotation("Second")
@MyAnnotation("Third")
class MyClass {}

// Read them
MyAnnotation[] annotations = MyClass.class.getAnnotationsByType(MyAnnotation.class);
for (MyAnnotation a : annotations) {
    System.out.println(a.value());
}
```

---

## Java 8 Coding Questions

### Q1: Filter, Transform, and Collect with Streams
Write a program that takes a list of integers, filters even numbers, squares them, and returns a list of results.

```java
public List<Integer> processNumbers(List<Integer> numbers) {
    return numbers.stream()
        .filter(n -> n % 2 == 0)
        .map(n -> n * n)
        .sorted()
        .collect(Collectors.toList());
}

// Test
List<Integer> input = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6);
List<Integer> result = processNumbers(input);
// [4, 16, 36]
```

### Q2: Word Frequency Counter
Given a string, count the frequency of each word using Streams and Collectors.

```java
public Map<String, Long> wordFrequency(String text) {
    return Arrays.stream(text.toLowerCase().split("\\s+"))
        .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
}

// Test
Map<String, Long> freq = wordFrequency("hello world hello java world hello");
// {hello=3, world=2, java=1}
```

### Q3: Find Palindromes Using Stream API
Filter palindrome strings from a list using Streams.

```java
public List<String> findPalindromes(List<String> words) {
    return words.stream()
        .filter(w -> w.equals(new StringBuilder(w).reverse().toString()))
        .collect(Collectors.toList());
}
```

### Q4: Calculate Average Using Collectors
Calculate the average, min, and max of a list of doubles using a single collector.

```java
public DoubleSummaryStatistics calculateStats(List<Double> values) {
    return values.stream()
        .collect(Collectors.summarizingDouble(Double::doubleValue));
}

// Usage: stats.getAverage(), stats.getMin(), stats.getMax(), stats.getCount()
```

### Q5: Date Difference Calculator
Given two dates, calculate the years, months, and days between them using java.time.

```java
public String dateDifference(LocalDate start, LocalDate end) {
    Period period = Period.between(start, end);
    return String.format("%d years, %d months, %d days",
        period.getYears(), period.getMonths(), period.getDays());
}

// Test
String diff = dateDifference(LocalDate.of(1990, 5, 15), LocalDate.now());
```

### Q6: Async Data Processing Pipeline
Simulate an async pipeline: fetch user → fetch orders → calculate total.

```java
public CompletableFuture<Double> calculateOrderTotal(String userId) {
    return CompletableFuture.supplyAsync(() -> fetchUser(userId))
        .thenCompose(user -> CompletableFuture.supplyAsync(() -> fetchOrders(user)))
        .thenApply(orders -> orders.stream()
            .mapToDouble(Order::getTotal)
            .sum());
}

User fetchUser(String id) { sleep(100); return new User(id); }
List<Order> fetchOrders(User user) { sleep(200); return List.of(new Order(50.0)); }
```

### Q7: Parallel Stream Performance Test
Compare sequential vs parallel stream performance for a CPU-intensive task.

```java
public void performanceComparison() {
    List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
        .boxed().collect(Collectors.toList());

    long startSeq = System.nanoTime();
    long sumSeq = numbers.stream()
        .mapToLong(Integer::longValue).sum();
    long timeSeq = System.nanoTime() - startSeq;

    long startPar = System.nanoTime();
    long sumPar = numbers.parallelStream()
        .mapToLong(Integer::longValue).sum();
    long timePar = System.nanoTime() - startPar;

    System.out.println("Sequential: " + timeSeq / 1_000_000 + "ms");
    System.out.println("Parallel: " + timePar / 1_000_000 + "ms");
}
```

### Q8: Custom Collector
Write a custom collector that joins strings with a delimiter (like Collectors.joining).

```java
public static Collector<String, StringBuilder, String> joinWith(String delimiter) {
    return Collector.of(
        StringBuilder::new,
        (sb, s) -> {
            if (sb.length() > 0) sb.append(delimiter);
            sb.append(s);
        },
        (sb1, sb2) -> {
            if (sb1.length() > 0 && sb2.length() > 0) sb1.append(delimiter);
            sb1.append(sb2);
            return sb1;
        },
        StringBuilder::toString
    );
}

// Usage: list.stream().collect(joinWith(", "))
```
