# Java 17 – New/Enhanced Features (LTS)

## 1. Sealed Classes & Interfaces (Finalized)

### Sealed Interface → Multiple Permitted Types
```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public final class Triangle implements Shape {
    private final double a, b, c;
    public Triangle(double a, double b, double c) {
        this.a = a; this.b = b; this.c = c;
    }
    // equals, hashCode, toString auto-generated? No — need record for that
}

// non-sealed subclass of Triangle
non-sealed class RightTriangle extends Triangle {
    public RightTriangle(double a, double b) {
        super(a, b, Math.sqrt(a * a + b * b));
    }
}
```

### Sealed Class Pattern
```java
// Sealed class with multiple permitted types
public sealed abstract class Vehicle
    permits Car, Truck, Motorcycle {}

public final class Car extends Vehicle {
    private final String make;
    private final int year;
    public Car(String make, int year) { this.make = make; this.year = year; }
}

public final class Truck extends Vehicle {
    private final double payloadCapacity;
    public Truck(double capacity) { this.payloadCapacity = capacity; }
}

public sealed interface TwoWheeler extends Vehicle permits Motorcycle {}

public final class Motorcycle extends TwoWheeler {
    private final int cc;
    public Motorcycle(int cc) { this.cc = cc; }
}
```

### Exhaustive Switch with Sealed Types (Java 17)
```java
public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> calculateTriangleArea(t);
    };
    // No default needed — compiler verifies exhaustiveness
}
```

---

## 2. Pattern Matching for switch (Preview in Java 17)

```java
// Extended pattern matching in switch (preview)
public String formatValue(Object obj) {
    return switch (obj) {
        case String s -> "String: " + s;
        case Integer i -> "Integer: " + i;
        case Long l -> "Long: " + l;
        case String[] arr -> "String array: " + Arrays.toString(arr);
        case int[] arr -> "int array: " + Arrays.toString(arr);
        case null -> "null";
        default -> obj.toString();
    };
}

// Deconstruction patterns (Java 21 final, preview in 17)
public String describe(Shape shape) {
    return switch (shape) {
        case Circle(var radius) -> "Circle(r=%s)".formatted(radius);
        case Rectangle(var w, var h) -> "Rectangle(%sx%s)".formatted(w, h);
        case Triangle(var a, var b, var c) -> "Triangle(%s,%s,%s)".formatted(a, b, c);
    };
}
```

---

## 3. Enhanced Pseudo-Random Number Generators (Java 17)

### New RNG Interfaces and Implementations
```java
// RandomGenerator interface (Java 17+)
RandomGenerator rng = RandomGenerator.of("LCG");  // Linear Congruential
RandomGenerator mt = RandomGenerator.of("MT19937"); // Mersenne Twister
RandomGenerator xoshiro = RandomGenerator.of("Xoshiro256StarStar");

// Using it
Random r = RandomGenerator.of("Xoshiro256StarStar");
int val = r.nextInt(100);

// GeneratorFactory
RandomGenerator.LegacyLegacyLegacyLegacyLegacy // Built-in options
```

### Types
- **LegacyLegacyLegacy**: Default Random (still uses LCG)
- **LXM**: XorShift/Mersenne Twister (default for SplittableRandom in 17+)
- **L128X1024StarStar**: High-quality, fast
- **BCrypt**: Bcrypt hashing (for password hashing)

```java
// Creating Random instances
Random random = RandomGenerator.of("LXM");
IntStream ints = random.ints(100, 0, 1000);

// SplittableRandom with new algorithm
SplittableRandom sr = new SplittableRandom();
sr.setGenerator("Xoshiro256StarStar");
```

---

## 4. Strong Encapsulation of JDK Internals (Module System Enforcement)

```java
// Java 17 enforces strong encapsulation by default
// Before Java 17: --add-opens was needed for reflection on JDK internals
// Java 17: JDK internal packages are inaccessible by default

// This will FAIL in Java 17 without --add-opens:
// Reflection on java.util.ArrayList internal fields
// Field f = ArrayList.class.getDeclaredField("elementData");
// f.setAccessible(true); // Illegal in Java 17 (strong encapsulation)

// Workaround: --add-opens flag
// java --add-opens java.base/java.util=ALL-UNNAMED MyApp
```

---

## 5. Deprecation/Removal of Legacy Tools

### Removed/Deprecated in Java 17
```java
// Applet API — removed
// Applet, JApplet removed (deprecated since Java 9)

// Security Manager — deprecated (for removal in future)
// System.setSecurityManager() deprecated
// SecurityManager class deprecated

// RMI Activation — removed
// java.rmi.activation package removed

// Harfango font renderer — removed
// Solaris/SPARC specific
```

---

## 6. Foreign Function & Memory API (Incubator in Java 17)

### Memory Access
```java
// Foreign Memory Access (incubator) — accessing native memory
// Layout describing a C struct
SequenceLayout C_INT = ValueLayout.JAVA_INT;
SequenceLayout C_DOUBLE = ValueLayout.JAVA_DOUBLE;

// Allocate native memory
try (Arena arena = Arena.ofConfined()) {
    MemorySegment segment = arena.allocate(1024);
    segment.set(ValueLayout.JAVA_INT, 0, 42);
    int value = segment.get(ValueLayout.JAVA_INT, 0);
}
```

### Linking Native Functions
```java
// Link to native library (simplified)
Linker linker = Linker.nativeLinker();
SymbolLookup libc = linker.defaultLookup();

// Call C's strlen from Java
MethodHandle strlen = linker.downcallHandle(
    libc.find("strlen").get(),
    FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
);

try (Arena arena = Arena.ofConfined()) {
    MemorySegment str = arena.allocateFrom("Hello");
    long length = (long) strlen.invoke(str);
}
```

---

## 7. macOS AArch64 (Apple Silicon) Support

```bash
# Java 17 runs natively on Apple Silicon (M1/M2)
# - No x86 emulation needed
# - Performance improvements on ARM architecture
java -version
# Should show: aarch64 on Apple Silicon Macs
```

---

## Java 17 Coding Questions

### Q1: Exhaustive Switch with Sealed Hierarchy
Process geometric shapes using sealed classes and exhaustive switch.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double a, double b, double c) implements Shape {
    public Triangle {
        if (a + b <= c || a + c <= b || b + c <= a) {
            throw new IllegalArgumentException("Invalid triangle sides");
        }
    }
}

public class ShapeProcessor {
    public double area(Shape shape) {
        return switch (shape) {
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.width() * r.height();
            case Triangle t -> {
                double s = (t.a() + t.b() + t.c()) / 2;
                yield Math.sqrt(s * (s - t.a()) * (s - t.b()) * (s - t.c()));
            }
        };
    }

    public String describe(Shape shape) {
        return switch (shape) {
            case Circle c -> String.format("Circle(r=%.2f)", c.radius());
            case Rectangle r -> String.format("Rectangle(%.2f x %.2f)", 
                r.width(), r.height());
            case Triangle t -> String.format("Triangle(sides: %.2f, %.2f, %.2f)",
                t.a(), t.b(), t.c());
        };
    }
    
    public List<Double> areas(List<Shape> shapes) {
        return shapes.stream()
            .map(this::area)
            .toList(); // Java 16+ unmodifiable list
    }
}
```

### Q2: Pattern Matching for instanceof
Process different request types using pattern matching.

```java
public sealed interface Request permits ReadRequest, WriteRequest, DeleteRequest {}
public record ReadRequest(String resource) implements Request {}
public record WriteRequest(String resource, String data) implements Request {}
public record DeleteRequest(String resource) implements Request {}

public class RequestHandler {
    public String handle(Request request) {
        if (request instanceof ReadRequest r) {
            return "Reading: " + r.resource();
        } else if (request instanceof WriteRequest w) {
            return "Writing to " + w.resource() + ": " + w.data();
        } else if (request instanceof DeleteRequest d) {
            return "Deleting: " + d.resource();
        }
        return "Unknown request";
    }
    
    // With guarded patterns (Java 17+)
    public String classify(Request request) {
        return switch (request) {
            case ReadRequest r when r.resource().startsWith("admin") -> "Admin read";
            case ReadRequest r -> "User read";
            case WriteRequest w when w.data().length() > 1000 -> "Large write";
            case WriteRequest w -> "Standard write";
            case DeleteRequest d -> "Delete";
        };
    }
}
```

### Q3: Enhanced RNG
Use the new Enhanced Pseudo-Random Number Generators.

```java
public class RngDemo {
    public static void main(String[] args) {
        // Generate with different algorithms
        Random fast = RandomGenerator.of("L128X1024StarStar");
        Random balanced = RandomGenerator.of("LXM");
        
        // Generate random integers
        IntStream.generate(() -> fast.nextInt(0, 100))
            .limit(10)
            .forEach(System.out::println);
        
        // SplittableRandom with new algorithm
        SplittableRandom sr = new SplittableRandom();
        sr.setGenerator("Xoshiro256StarStar");
        
        // Parallel random generation
        int[] randoms = sr.ints(1000, 1, 101).parallel().toArray();
        double avg = Arrays.stream(randoms).average().orElse(0);
        System.out.println("Average: " + avg);
    }
}
```

### Q4: Sealed Class Enum Processor
Process user roles using sealed class hierarchy and switch.

```java
public sealed interface Role permits Admin, User, Guest {}
public record Admin(String username, List<String> permissions) implements Role {}
public record User(String username, String email) implements Role {}
public record Guest(String sessionId) implements Role {}

public class AccessControl {
    public boolean canAccess(Role role, String resource) {
        return switch (role) {
            case Admin a -> true; // admins can access everything
            case User u -> resource.startsWith("public/") 
                || resource.startsWith("user/" + u.username());
            case Guest g -> resource.startsWith("public/");
        };
    }
    
    public List<String> getPermissions(Role role) {
        return switch (role) {
            case Admin a -> a.permissions();
            case User u -> List.of("read", "write_own");
            case Guest g -> List.of("read_public");
        };
    }
}
```

### Q5: Native Memory Operations
Perform basic native memory operations using Foreign Function & Memory API.

```java
public class NativeDemo {
    public static void main(String[] args) {
        try (Arena arena = Arena.ofConfined()) {
            // Allocate and initialize native memory
            MemorySegment nameSegment = arena.allocateFrom("Hello");
            MemorySegment numbers = arena.allocateArray(ValueLayout.JAVA_INT, 5);
            
            // Set values
            numbers.set(ValueLayout.JAVA_INT, 0, 10);
            numbers.set(ValueLayout.JAVA_INT, 1, 20);
            numbers.set(ValueLayout.JAVA_INT, 2, 30);
            
            // Read values
            int sum = 0;
            for (int i = 0; i < 5; i++) {
                sum += numbers.get(ValueLayout.JAVA_INT, i);
            }
            System.out.println("Sum of first 3: " + 
                (numbers.get(ValueLayout.JAVA_INT, 0) +
                 numbers.get(ValueLayout.JAVA_INT, 1) +
                 numbers.get(ValueLayout.JAVA_INT, 2)));
            
            // Memory copy
            MemorySegment copy = arena.allocate(10);
            copy.copyFrom(numbers.asSlice(0, 4));
        }
    }
}
```
