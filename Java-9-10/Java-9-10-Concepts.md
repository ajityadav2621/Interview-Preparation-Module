# Java 9–10 – Notable Additions

## 1. Java Platform Module System (JPMS / Jigsaw)

### Module Declaration
```java
// module-info.java
module com.myapp {
    // Requires other modules
    requires java.base;
    requires java.desktop;
    requires transitive com.library; // transitive dependency
    
    // Exports packages
    exports com.myapp.api;
    
    // Opens for reflection (needed for frameworks like Jackson, Hibernate)
    opens com.myapp.internal to com.fasterxml.jackson;
    
    // Uses a service
    uses com.myapp.spi.Plugin;
    
    // Provides a service implementation
    provides com.myapp.spi.Plugin with com.myapp.impl.DefaultPlugin;
}
```

### Module Types
```java
// Named module
module com.myapp { ... }

// Automatic module (JAR on classpath becomes module)
// No module-info.java — JAR filename becomes module name

// Unnamed module (everything on classpath)
// All unnamed modules read all named modules
```

### Access Rules
```java
// Exported packages are accessible by other modules
// Non-exported packages are module-private
// opens = accessible via reflection
// open = all packages open at runtime
```

---

## 2. JShell (REPL)

```bash
# Launch: jshell
# Commands within JShell:
/help                          # Show help
/list                           # List entered snippets
/save myfile.java              # Save history
/open myfile.java              # Open file
/eval expression               # Evaluate
/exit                           # Exit
```

### Usage Examples
```java
jshell> int x = 10;
x ==> 10

jshell> x * 2
$2 ==> 20

jshell> void greet(String name) { ... }
|  created method greet(String)

jshell> /import java.util.stream.*

jshell> List<String> names = List.of("A", "B", "C");
names ==> [A, B, C]
```

---

## 3. Private Methods in Interfaces

```java
interface DataProcessor {
    // Public abstract method
    void process(Data data);
    
    // Default method using private helper
    default void processAndLog(Data data) {
        process(data);
        log(data);
    }
    
    // Private method — only accessible within this interface
    private void log(Data data) {
        System.out.println("Processed: " + data);
    }
    
    // Static private method (Java 9+)
    private static String formatTimestamp(long timestamp) {
        return Instant.ofEpochMilli(timestamp).toString();
    }
}
```

---

## 4. var for Local Type Inference (Java 10)

```java
// var infers type at compile time
var list = new ArrayList<String>();           // ArrayList<String>
var map = new HashMap<String, Integer>();      // HashMap<String, Integer>
var stream = IntStream.range(1, 100);          // IntStream
var today = LocalDate.now();                   // LocalDate
var entry = map.entrySet().iterator().next();  // Map.Entry<String,Integer>

// var CANNOT be used for:
// - Fields (class members)
// - Method parameters
// - Method return types
// - Lambda parameters (before Java 11)
// - Multiple declarations: var a, b = 1; // ERROR

// var with try-with-resources (Java 10+)
try (var reader = Files.newBufferedReader(path)) {
    // ...
}
```

### Restrictions
```java
// var requires initializer
var x;           // ERROR: cannot infer type without initializer
// var y = null; // ERROR: cannot infer type from null
// var z;        // ERROR

// var with array creation not allowed directly
// var arr = new int[10]; // ERROR in Java 10 (allowed in later versions via varargs trick)
```

---

## 5. Improved Stream API (Java 9)

### takeWhile — take elements while predicate is true
```java
List<Integer> taken = Stream.of(1, 2, 3, 4, 5, 3, 2, 1)
    .takeWhile(n -> n < 4)
    .collect(Collectors.toList());
// [1, 2, 3] — stops at first element that doesn't match
```

### dropWhile — skip elements while predicate is true
```java
List<Integer> dropped = Stream.of(1, 2, 3, 4, 5, 3, 2, 1)
    .dropWhile(n -> n < 4)
    .collect(Collectors.toList());
// [4, 5, 3, 2, 1] — drops from start until predicate fails
```

### ofNullable — single element stream or empty
```java
Stream<String> s1 = Stream.ofNullable("value");  // [value]
Stream<String> s2 = Stream.ofNullable(null);     // [] (empty)
```

### Iterable.forEachRemaining
```java
// More efficient for Spliterators
List<Integer> list = List.of(1, 2, 3);
list.forEachRemaining(System.out::println);
```

---

## 6. Immutable Collection Factory Methods (Java 9+)

```java
// List
List<String> list = List.of("A", "B", "C");
List<String> empty = List.of(); // or List.copyOf(otherList)

// Set
Set<String> set = Set.of("A", "B", "C");

// Map
Map<String, Integer> map = Map.of(
    "one", 1,
    "two", 2,
    "three", 3
);
Map<String, Integer> map2 = Map.ofEntries(
    Map.entry("one", 1),
    Map.entry("two", 2)
);

// Properties (Java 10+)
Properties props = Map.of("key", "value").entrySet().stream()
    .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().toString(), (a, b) -> a, Properties::new));

// Rules:
// - No null elements/keys (throws NullPointerException)
// - Immutable (no add/remove)
// - Max 10 entries for of() (use Map.ofEntries for more)
```

### Copy Of (Java 10+)
```java
List<String> copy = List.copyOf(originalList); // Unmodifiable copy
Set<String> copySet = Set.copyOf(originalSet);
Map<String, Integer> copyMap = Map.copyOf(originalMap);
```

---

## 7. Try-with-Resources Enhancement (Java 9+)

```java
// Before Java 9 — resource must be final/effectively final
final BufferedReader br1 = new BufferedReader(new FileReader("a.txt"));
final BufferedReader br2 = new BufferedReader(new FileReader("b.txt"));
try (br1; br2) {
    // use both
}

// Java 9+ — effectively final works too
var br1 = new BufferedReader(new FileReader("a.txt"));
var br2 = new BufferedReader(new FileReader("b.txt"));
try (br1; br2) {
    // use both — no need for final modifier
}
```

---

## 8. Other Java 9–10 Features

### Process API (Java 9+)
```java
ProcessHandle.current().pid();
ProcessHandle.allProcesses().forEach(p -> 
    System.out.println(p.pid() + " " + p.info().command().orElse("?")));
```

### HTTP Client (Java 9/11)
```java
// Java 9 (incubating) — java.net.http
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .GET()
    .build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
```

### Multi-Release JARs
```
META-INF/versions/9/module-info.java  — Java 9+ specific
META-INF/versions/10/module-info.java — Java 10+ specific
```

### Deprecate for Removal
```java
@Deprecated(since = "10", forRemoval = true)
public void oldMethod() { ... }
```
