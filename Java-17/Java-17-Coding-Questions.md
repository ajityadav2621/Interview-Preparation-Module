# Java 17 – Coding Questions

## Q1: Sealed Class Shape Processor
Build a shape processor using Java 17 sealed classes and exhaustive switch.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
    public double perimeter() { return 2 * Math.PI * radius; }
}

public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
    public double perimeter() { return 2 * (width + height); }
}

public final class Triangle implements Shape {
    private final double a, b, c;
    public Triangle(double a, double b, double c) {
        if (a + b <= c || a + c <= b || b + c <= a) {
            throw new IllegalArgumentException("Invalid triangle");
        }
        this.a = a; this.b = b; this.c = c;
    }
    public double area() {
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
    public double perimeter() { return a + b + c; }
}

public class ShapeService {
    public double totalArea(List<Shape> shapes) {
        return shapes.stream()
            .mapToDouble(this::computeArea)
            .sum();
    }
    
    private double computeArea(Shape s) {
        return switch (s) {
            case Circle c -> c.area();
            case Rectangle r -> r.area();
            case Triangle t -> t.area();
        };
    }
    
    public Map<String, Double> areaByType(List<Shape> shapes) {
        return shapes.stream()
            .collect(Collectors.groupingBy(
                s -> switch(s) {
                    case Circle c -> "Circle";
                    case Rectangle r -> "Rectangle";
                    case Triangle t -> "Triangle";
                },
                Collectors.summingDouble(this::computeArea)
            ));
    }
}
```

## Q2: Pattern Matching Request Router
Route HTTP-like requests using pattern matching.

```java
public sealed interface Request permits GetRequest, PostRequest, DeleteRequest {}
public record GetRequest(String path, Map<String, String> params) implements Request {}
public record PostRequest(String path, String body) implements Request {}
public record DeleteRequest(String path) implements Request {}

public class RequestRouter {
    public String route(Request request) {
        if (request instanceof GetRequest g) {
            return "GET " + g.path() + " params=" + g.params();
        } else if (request instanceof PostRequest p) {
            return "POST " + p.path() + " body=" + p.body();
        } else if (request instanceof DeleteRequest d) {
            return "DELETE " + d.path();
        }
        return "Unknown";
    }
    
    // Java 17 guarded patterns
    public String categorize(Request request) {
        return switch (request) {
            case GetRequest g when g.path().startsWith("/api/v2") -> "API v2 GET";
            case GetRequest g -> "GET";
            case PostRequest p when p.body().length() > 10_000 -> "Large POST";
            case PostRequest p -> "POST";
            case DeleteRequest d -> "DELETE";
        };
    }
}
```

## Q3: Random Number Generator Benchmark
Benchmark different RNG algorithms.

```java
public class RngBenchmark {
    public static void main(String[] args) {
        String[] algorithms = { "LXM", "L128X1024StarStar", "Xoshiro256StarStar", "PCG32" };
        
        for (String algo : algorithms) {
            Random rng = RandomGenerator.of(algo);
            long start = System.nanoTime();
            long count = IntStream.generate(() -> rng.nextInt())
                .limit(10_000_000)
                .filter(i -> i > 0)
                .count();
            long time = System.nanoTime() - start;
            System.out.printf("%s: %d positive values in %d ms%n",
                algo, count, time / 1_000_000);
        }
    }
}
```

## Q4: Role-Based Access Control with Sealed Types
Implement RBAC using sealed interfaces and pattern matching.

```java
public sealed interface Permission permits ReadPerm, WritePerm, AdminPerm {}
public record ReadPerm(String resource) implements Permission {}
public record WritePerm(String resource) implements Permission {}
public record AdminPerm() implements Permission {}

public sealed interface UserType permits StandardUser, PrivilegedUser {}
public record StandardUser(String id, List<Permission> perms) implements UserType {}
public record PrivilegedUser(String id, List<Permission> perms, String department) 
    implements UserType {}

public class AccessControl {
    public boolean hasPermission(UserType user, Permission required) {
        return switch (user) {
            case StandardUser u -> u.perms().contains(required);
            case PrivilegedUser p -> {
                if (required instanceof AdminPerm) yield false;
                yield p.perms().contains(required);
            }
        };
    }
}
```

## Q5: Native Memory Data Structure
Implement a simple structure in native memory.

```java
public class NativeStructDemo {
    public record Person(String name, int age, double salary) {}
    
    public static void main(String[] args) {
        try (Arena arena = Arena.ofConfined()) {
            // Allocate struct-like memory
            ValueLayout.OfInt ageLayout = ValueLayout.JAVA_INT;
            ValueLayout.OfDouble salaryLayout = ValueLayout.JAVA_DOUBLE;
            
            // Allocate name (128 bytes for string) + age + salary
            MemorySegment person = arena.allocate(128 + ageLayout.byteSize() 
                + salaryLayout.byteSize());
            
            // Store age and salary at known offsets
            person.set(ageLayout, 128, 30);
            person.set(salaryLayout, 128 + 4, 75000.0);
            
            // Retrieve
            int age = person.get(ageLayout, 128);
            double salary = person.get(salaryLayout, 128 + 4);
            System.out.println("Age: " + age + ", Salary: " + salary);
        }
    }
}
```
