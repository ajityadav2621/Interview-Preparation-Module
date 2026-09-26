# Java 12–16 – Coding Questions

## Q1: Switch Expression Calculator
Implement a calculator using switch expression with yield for complex operations.

```java
public class Calculator {
    public double calculate(double a, double b, String op) {
        return switch (op) {
            case "+" -> a + b;
            case "-" -> a - b;
            case "*" -> a * b;
            case "/" -> {
                if (b == 0) yield Double.NaN;
                else yield a / b;
            }
            case "mod" -> {
                yield a - (Math.floor(a / b) * b);
            }
            case "pow" -> Math.pow(a, b);
            default -> throw new IllegalArgumentException("Unknown: " + op);
        };
    }
}
```

## Q2: Text Block Config Parser
Parse a configuration text block into a map.

```java
public Map<String, String> parseConfig(String config) {
    return config.strip().lines()
        .filter(line -> !line.strip().startsWith("#"))
        .filter(line -> line.contains("="))
        .map(line -> line.strip().split("=", 2))
        .collect(Collectors.toMap(
            parts -> parts[0].strip(),
            parts -> parts.length > 1 ? parts[1].strip() : ""
        ));
}

// Usage
String config = """
    # Server config
    host=localhost
    port=8080
    debug=true
    """;
Map<String, String> cfg = parseConfig(config);
```

## Q3: Shape Hierarchy with Sealed Classes
Create a complete shape system with area and perimeter calculations.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
    public double perimeter() { return 2 * Math.PI * radius; }
}

public record Rectangle(double w, double h) implements Shape {
    public double area() { return w * h; }
    public double perimeter() { return 2 * (w + h); }
}

public record Triangle(double a, double b, double c) implements Shape {
    public double area() {
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
    public double perimeter() { return a + b + c; }
}

public class ShapeAnalyzer {
    public static Map<String, Double> describe(Shape shape) {
        return switch (shape) {
            case Circle c -> Map.of(
                "area", c.area(),
                "perimeter", c.perimeter(),
                "type", "Circle");
            case Rectangle r -> Map.of(
                "area", r.area(),
                "perimeter", r.perimeter(),
                "type", "Rectangle");
            case Triangle t -> Map.of(
                "area", t.area(),
                "perimeter", t.perimeter(),
                "type", "Triangle");
        };
    }
}
```

## Q4: Record-based Student Grading System
Use records for student data and grading.

```java
public record Student(String id, String name, List<Integer> scores) {
    public Student {
        if (scores == null || scores.isEmpty()) {
            scores = List.of();
        }
    }
    
    public double average() {
        return scores.stream().mapToInt(Integer::intValue).average().orElse(0.0);
    }
    
    public String grade() {
        double avg = average();
        return switch {
            case double a when a >= 90 -> "A";
            case double a when a >= 80 -> "B";
            case double a when a >= 70 -> "C";
            case double a when a >= 60 -> "D";
            default -> "F";
        };
    }
}

// Usage: new Student("001", "Alice", List.of(90, 85, 92)).grade()
```

## Q5: instanceof Pattern Matching Formatter
Format different object types using pattern matching.

```java
public String formatValue(Object value) {
    if (value == null) return "null";
    if (value instanceof String s) return "\"" + s + "\"";
    if (value instanceof byte[] bytes) return "byte[" + bytes.length + "]";
    if (value instanceof Number n) return n.toString();
    if (value instanceof List<?> list) return list.toString();
    if (value instanceof Map<?, ?> map) return map.toString();
    return value.toString();
}
```

## Q6: Text Block Template Engine
Build a simple template engine using text blocks and String.formatted().

```java
public class TemplateEngine {
    private static final String TEMPLATE = """
        Dear %s,
        
        Thank you for your order #%s.
        Your total is $%.2f.
        
        Estimated delivery: %s
        
        Best regards,
        %s
        """;
    
    public String generate(String customer, String orderId, 
                           double total, String date, String company) {
        return TEMPLATE.formatted(customer, orderId, total, date, company);
    }
}
```

## Q7: Exhaustive Switch on Sealed Hierarchy
Process payment methods using sealed classes and exhaustive switch.

```java
public sealed interface PaymentMethod permits CreditCard, PayPal, Crypto {}

public record CreditCard(String number, String cvv) implements PaymentMethod {}
public record PayPal(String email) implements PaymentMethod {}
public record Crypto(String walletAddress, String network) implements PaymentMethod {}

public class PaymentProcessor {
    public String process(PaymentMethod method) {
        return switch (method) {
            case CreditCard cc -> "Processing $" + cc.number().length() + " digit card";
            case PayPal pp -> "Processing PayPal: " + pp.email();
            case Crypto cr -> "Processing " + cr.network() + " crypto";
        };
    }
    
    // Additional: pattern matching + guarded patterns (Java 17+)
    public String categorize(PaymentMethod method) {
        return switch (method) {
            case CreditCard cc when cc.number().startsWith("4") -> "Visa card";
            case CreditCard cc -> "Other card";
            case PayPal pp -> "PayPal";
            case Crypto cr when "Bitcoin".equals(cr.network()) -> "Bitcoin";
            case Crypto cr -> "Other crypto";
        };
    }
}
```

## Q8: Record Deep Copy and Conversion
Convert between entity and record, with validation and transformation.

```java
public record ProductDto(String name, double price, String category) {
    public ProductDto {
        if (price < 0) throw new IllegalArgumentException("Price < 0");
    }
}

public class ProductConverter {
    public static ProductDto toDto(ProductEntity entity) {
        if (entity instanceof ProductEntity e) {
            return new ProductDto(
                e.getName().strip().toUpperCase(),
                Math.round(e.getPrice() * 100.0) / 100.0,
                e.getCategory() != null ? e.getCategory() : "UNCATEGORIZED"
            );
        }
        throw new IllegalArgumentException("Invalid entity");
    }
    
    public static List<ProductDto> toDtoList(List<ProductEntity> entities) {
        return entities.stream().map(ProductConverter::toDto).collect(Collectors.toList());
    }
}
```
