# Java 12–16 – Notable Additions

## 1. Switch Expressions (Java 12 Preview → 14 Final)

### Arrow Syntax (->)
```java
// Traditional switch
String dayType;
switch (day) {
    case "MON": case "TUE": case "WED":
    case "THU": case "FRI":
        dayType = "Weekday";
        break;
    case "SAT": case "SUN":
        dayType = "Weekend";
        break;
    default:
        throw new IllegalArgumentException("Unknown day: " + day);
}

// Switch expression (Java 14+)
String dayType = switch (day) {
    case "MON", "TUE", "WED", "THU", "FRI" -> "Weekday";
    case "SAT", "SUN" -> "Weekend";
    default -> throw new IllegalArgumentException("Unknown day: " + day);
};
```

### Yield Keyword (Java 14+)
```java
int numLetters = switch (day) {
    case "MON" -> {
        System.out.println("Monday");
        yield 5; // returns value from block
    }
    case "TUE" -> {
        System.out.println("Tuesday");
        yield 5;
    }
    default -> throw new IllegalArgumentException();
};

// Yield replaces return in switch expressions
// yield sends a value back to the switch expression
```

### Switch as Expression/Statement/Statement Block
```java
// As expression (yield value)
int x = switch (s) { case "a" -> 1; default -> 0; };

// As statement (just execute)
switch (s) {
    case "a" -> System.out.println("A");
    default -> System.out.println("Other");
}

// As code block (curly braces)
{
    switch (s) {
        case "a" -> { block1(); block2(); }
        default -> block3();
    }
}
```

---

## 2. Text Blocks (Java 13 Preview → 15 Final)

### Basic Syntax
```java
// Old way (string concatenation)
String json = "{\n"
    + "  \"name\": \"John\",\n"
    + "  \"age\": 30\n"
    + "}";

// Text block (Java 15+)
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;
```

### Processing Text Blocks
```java
String html = """
    <html>
        <body>
            <p>Hello, World!</p>
        </body>
    </html>
    """;

// Common operations:
String trimmed = html.strip();              // Remove leading/trailing whitespace
String escaped = html.translateEscapes();   // Process escape sequences (\n, \t, etc.)
String indented = html.indent(4);           // Add 4 spaces to each line
String uninherited = html.stripIndent();    // Remove common leading whitespace

// Text block with formatting
String query = """
    SELECT %s, %s, %s
    FROM %s
    WHERE id = %d
    """;
String sql = String.formatted(query, "name", "age", "email", "users", 42);
// Or: String sql = query.formatted("name", "age", "email", "users", 42);
```

### Text Block Indentation Rules
```java
// The opening """ can be followed by newline
// Content starts on next line
// Closing """ can be at column 0 or indented

// Common pattern:
String sql = """
        SELECT *
        FROM users
        WHERE active = true
    """; // Closing at column 0 → common whitespace removed
// Result: SELECT *\nFROM users\nWHERE active = true
```

---

## 3. Records (Java 14 Preview → 16 Final)

### Basic Record
```java
// Record = immutable data carrier with auto-generated:
// - constructor
// - getters (field access, not getX())
// - equals() / hashCode()
// - toString()

public record Point(int x, int y) {}

Point p = new Point(3, 4);
p.x();  // 3 (auto-generated accessor)
p.y();  // 4
p.toString(); // "Point[x=3, y=4]"
p.hashCode(); // auto-generated
// p.x = 5; // ERROR — fields are final
```

### Compact Constructor
```java
public record Person(String name, int age) {
    // Validation in compact constructor
    public Person {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
    }
}
```

### Custom Methods
```java
public record Circle(double radius) {
    // Extra method
    public double area() {
        return Math.PI * radius * radius;
    }
    
    // Custom toString
    @Override
    public String toString() {
        return String.format("Circle(r=%.2f)", radius);
    }
}
```

### Static Factory Method
```java
public record Point(int x, int y) {
    public static Point origin() {
        return new Point(0, 0);
    }
    
    public Point translate(int dx, int dy) {
        return new Point(x + dx, y + dy);
    }
}
```

### Record with Generic Types
```java
public record ApiResponse<T>(
    int status,
    String message,
    T data
) {}

ApiResponse<User> response = new ApiResponse<>(200, "OK", user);
```

### Non-static Fields (Rare, advanced)
```java
public record Counter(int id) {
    private int count = 0; // Non-static, non-final — manual management
    
    public void increment() { count++; }
    public int getCount() { return count; }
}
```

---

## 4. Pattern Matching for instanceof (Java 14 Preview → 16 Final)

### Before (Manual casting)
```java
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.toUpperCase());
}
```

### After (Pattern matching)
```java
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // s is already cast
}

// With else
if (obj instanceof String s) {
    System.out.println("String: " + s);
} else if (obj instanceof Integer i) {
    System.out.println("Integer: " + i);
} else {
    System.out.println("Unknown");
}

// In switch (Java 17+ preview/19+ final)
String format(Object obj) {
    return switch (obj) {
        case String s -> "String: " + s;
        case Integer i -> "Integer: " + i;
        case null -> "null";
        default -> "Unknown";
    };
}
```

### Pattern Matching for null check
```java
// Pattern matching handles null gracefully (instanceof returns false)
Object obj = null;
if (obj instanceof String s) {
    // Never reached for null
} else {
    System.out.println("Not a string or null");
}
```

---

## 5. Sealed Classes (Java 15 Preview → Java 17 Final)

### Basic Sealed Class
```java
// Sealed — restricted to permitted subclasses
public sealed abstract class Shape
    permits Circle, Rectangle, Triangle {}

// Permitted subclasses can be final or sealed
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public sealed interface Polygon extends Shape
    permits Triangle, Square {}

// Non-sealed (default) — can be extended by anyone
public non-sealed class Square implements Polygon {
    private final double side; // Requires manual equals/hashCode
    public Square(double side) { this.side = side; }
}

public final class Triangle implements Polygon {
    private final double a, b, c;
    public Triangle(double a, double b, double c) {
        this.a = a; this.b = b; this.c = c;
    }
}
```

### Exhaustive Switch (Java 17+)
```java
public double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> {
            double s = (t.a() + t.b() + t.c()) / 2;
            yield Math.sqrt(s * (s - t.a()) * (s - t.b()) * (s - t.c()));
        }
        // No default needed — compiler knows all subtypes (if sealed)
    };
}
```

### Sealed Class + Pattern Matching
```java
public String describe(Shape shape) {
    return switch (shape) {
        case Circle c when c.radius() > 10 -> "Large circle";
        case Circle c -> "Small circle";
        case Rectangle r -> "Rectangle";
        case null -> "No shape";
    };
}
```

---

## 6. Helpful NullPointerExceptions (Java 14)

### Before
```java
String name = null;
int len = name.length(); // NullPointerException — but WHICH variable?
// Exception in thread "main" java.lang.NullPointerException
```

### After (Java 14+)
```java
String name = null;
int len = name.length();
// Exception in thread "main" java.lang.NullPointerException:
//   Cannot invoke "String.length()" because "name" is null
```

### Multi-variable helpful NPE
```java
// For complex expressions
String result = user.getAddress().getCity().toUpperCase();
// Exception points to the specific reference:
// "Cannot invoke "String.toUpperCase()" because the return value of
//  "Address.getCity()" is null"
```

### Enable Helpful NPE (if needed)
```bash
java -XX:+ShowCodeDetailsInExceptionMessages MyApp
```

---

## 7. String.formatted() (Java 15+)

```java
String name = "John";
int age = 30;

// Before
String message = String.format("Name: %s, Age: %d", name, age);

// After (Java 15+)
String message = "Name: %s, Age: %d".formatted(name, age);

// More readable, especially with text blocks
String query = """
    SELECT * FROM users
    WHERE name = '%s' AND age > %d
    """.formatted(name, age);
```

---

## 8. Files.mismatch() (Java 12+)

```java
// Find first mismatched byte position between two files
long mismatch = Files.mismatch(path1, path2);
if (mismatch == -1) {
    System.out.println("Files are identical");
} else {
    System.out.println("First difference at position: " + mismatch);
}
```

---

## Java 12–16 Coding Questions

### Q1: Switch Expression with Yield
Use switch expression with yield for a calculator.

```java
public double calculate(double a, double b, String operator) {
    return switch (operator) {
        case "+" -> a + b;
        case "-" -> a - b;
        case "*" -> a * b;
        case "/" -> {
            if (b == 0) {
                yield Double.NaN;
            } else {
                yield a / b;
            }
        }
        default -> throw new IllegalArgumentException(
            "Unknown operator: " + operator);
    };
}
```

### Q2: Text Block JSON Builder
Build JSON using text blocks and String.formatted().

```java
public String buildPersonJson(String name, int age, String email) {
    return """
        {
          "name": "%s",
          "age": %d,
          "email": "%s"
        }
        """.formatted(name, age, email);
}
```

### Q3: Shape Area Calculator with Sealed Classes
Implement area calculation using sealed class hierarchy.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

public record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}

public record Triangle(double a, double b, double c) implements Shape {
    public double area() {
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
}

// Client code
public double totalArea(List<Shape> shapes) {
    return shapes.stream()
        .mapToDouble(shape -> switch (shape) {
            case Circle c -> c.area();
            case Rectangle r -> r.area();
            case Triangle t -> t.area();
        })
        .sum();
}
```

### Q4: Record Validation
Create a validated record for user registration.

```java
public record UserRegistration(
    String username,
    String email,
    int age,
    String password
) {
    public UserRegistration {
        if (username == null || username.isBlank() || username.length() < 3) {
            throw new IllegalArgumentException("Username must be 3+ chars");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        if (age < 13 || age > 120) {
            throw new IllegalArgumentException("Age must be 13-120");
        }
        if (password == null || password.length() < 8) {
            throw new IllegalArgumentException("Password must be 8+ chars");
        }
    }
    
    // Custom method
    public boolean isAdult() {
        return age >= 18;
    }
}
```

### Q5: Pattern Matching Demo
Use pattern matching for instanceof in a message processor.

```java
public String processMessage(Object message) {
    if (message instanceof String s) {
        return "Text message: " + s.toUpperCase();
    } else if (message instanceof List<?> list) {
        return "List with " + list.size() + " items";
    } else if (message instanceof Integer i && i > 100) {
        return "Large number: " + i;
    } else if (message instanceof Integer i) {
        return "Small number: " + i;
    } else if (message == null) {
        return "No message";
    } else {
        return "Unknown: " + message.getClass().getSimpleName();
    }
}
```

### Q6: Enhanced Switch with Pattern Matching
Combine switch expressions with pattern matching.

```java
public String describe(Object obj) {
    return switch (obj) {
        case String s when s.length() > 10 -> "Long string: " + s;
        case String s -> "Short string: " + s;
        case Integer i when i > 0 -> "Positive: " + i;
        case Integer i when i < 0 -> "Negative: " + i;
        case Integer i -> "Zero";
        case List<?> list -> "List of " + list.size();
        case Map<?, ?> map -> "Map with " + map.size() + " entries";
        case null -> "Null value";
        default -> "Unknown type";
    };
}
```

### Q7: Text Block SQL Query
Build a dynamic SQL query using text blocks.

```java
public String buildQuery(String table, List<String> columns, String whereClause) {
    String cols = columns.stream().collect(Collectors.joining(", "));
    String where = (whereClause != null && !whereClause.isBlank())
        ? "WHERE " + whereClause : "";
    
    return """
        SELECT %s
        FROM %s
        %s
        """
        .formatted(cols, table, where);
}
```

### Q8: Record with DTO Conversion
Convert between entities and records using pattern matching.

```java
public record UserDto(String name, String email) {}

public class UserEntity {
    private String name;
    private String email;
    private String passwordHash;
    // getters/setters...
}

public UserDto toDto(UserEntity entity) {
    if (entity instanceof UserEntity e) {
        return new UserDto(e.getName(), e.getEmail());
    }
    throw new IllegalArgumentException("Not a UserEntity");
}

// Record's natural fit for DTOs — auto equals/hashCode/toString
```
