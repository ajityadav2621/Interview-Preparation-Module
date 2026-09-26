# Java 11 – New/Enhanced Features

## 1. New String Methods

### isBlank() — checks if string is empty or only whitespace
```java
String s1 = "";         // true
String s2 = "   \t\n";  // true
String s3 = "hello";     // false
String s4 = "  hi  ";    // false (has non-whitespace)

s1.isBlank(); // true
s3.isBlank(); // false
```

### strip() — removes leading/trailing whitespace (Unicode-aware)
```java
String s = "  Hello World  ";
s.trim();   // "Hello World" — removes ASCII spaces, tabs, newlines
s.strip();  // "Hello World" — removes all Unicode whitespace

// strip() is preferred over trim() because:
// - It uses Character.isWhitespace() (Unicode-aware)
// - strip() removes characters like \u200B (zero-width space) that trim() doesn't
// - stripIndent() for multiline strings (Java 15+)
```

### lines() — returns stream of lines
```java
String multiline = "Line 1\nLine 2\nLine 3";
long count = multiline.lines().count(); // 3
List<String> lineList = multiline.lines().collect(Collectors.toList());
// [Line 1, Line 2, Line 3]

// Empty string → empty stream
"".lines().count(); // 0

// Trailing newline behavior:
"a\nb\n".lines().collect(Collectors.toList()); // [a, b]
```

### repeat(int count) — repeat string
```java
String dashes = "-".repeat(20); // --------------------
String pattern = "abc".repeat(3); // abcabcabc
String empty = "x".repeat(0); // ""
```

## 2. var in Lambda Parameters (Java 11)

```java
// Before Java 11 — var in lambdas was not allowed
// Java 11 — you can use var in lambda parameters

UnaryOperator<Integer> square = (var x) -> x * x;
BiFunction<String, String, String> concat = (var a, var b) -> a + b;

// Note: either ALL parameters must be var or NONE (in a lambda)
// (var a, b) -> ... // ERROR — inconsistent
```

## 3. New HttpClient API (Standardized in Java 11)

### Basic Usage
```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .GET()
    .build();

// Synchronous
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
int status = response.statusCode();
String body = response.body();

// Asynchronous
CompletableFuture<HttpResponse<String>> future = client.sendAsync(request, HttpResponse.BodyHandlers.ofString());
HttpResponse<String> asyncResponse = future.join();
```

### POST with JSON
```java
String jsonBody = "{\"name\":\"John\",\"age\":30}";
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
    .build();
```

### Streaming Response Body
```java
HttpResponse<Void> response = client.send(request, HttpResponse.BodyHandlers.fromLineSubscriber(
    BodySubscriber.create(line -> System.out.println("Line: " + line))
));
```

### WebSocket (Java 11+)
```java
WebSocketClient wsClient = WebSocketClient.create();
wsClient.connect(URI.create("ws://echo.websocket.org"))
    .thenAccept(ws -> {
        ws.sendText("Hello");
        ws.onText(message -> System.out.println("Echo: " + message));
    });
```

## 4. Files.readString() / Files.writeString()

```java
// Read entire file as string
String content = Files.readString(path);
String contentUtf8 = Files.readString(path, StandardCharsets.UTF_8);

// Write string to file
Files.writeString(path, "Hello World");
Files.writeString(path, content, StandardCharsets.UTF_8);

// With options
Files.writeString(path, content,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING,
    StandardOpenOption.WRITE);

// Before Java 11:
// String content = new String(Files.readAllBytes(path));
// Files.write(path, content.getBytes());
```

## 5. Running Single-File Source Code (Java 11+)

```bash
# No compilation needed — run directly
java HelloWorld.java

# Works for:
# - Single file programs
# - Shebang scripts (#!/usr/bin/java --source 11)
# - Multi-file source code (uses --source option)

# Shebang example:
#!/usr/bin/java --source 11
public class Script {
    public static void main(String[] args) {
        System.out.println("Hello from script!");
    }
}
```

## 6. Nest-Based Access Control (Java 11)

```java
// Before Java 11: compiler generated synthetic accessor methods for nested class access
// Java 11+: JVM supports nest members natively

public class Outer {
    private int secret = 42;
    
    class Inner {
        void access() {
            System.out.println(secret); // Direct access, no synthetic method
        }
    }
    
    // Runtime inspection
    public static void main(String[] args) {
        Class<?> outer = Outer.class;
        System.out.println("Nest host: " + outer.getNestHost());
        System.out.println("Nest members: " + Arrays.toString(outer.getNestMembers()));
    }
}
```

## 7. Epsilon GC (Java 11+)

```bash
# No-op garbage collector — useful for performance testing
# Memory allocation happens but nothing is freed
java -XX:+UseEpsilonGC MyApp

# Use cases:
# - Short-lived applications
# - Memory profiling (isolate heap pressure from GC effects)
# - Testing OOM behavior
```

## 8. Flight Recorder (Open-Sourced Java 11)

```bash
# Java Flight Recorder (JFR) — production profiling
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp

# JCMD:
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# JFR in JDK Mission Control:
# Analyze recordings for:
# - CPU hotspots
# - Memory allocation patterns
# - Lock contention
# - GC activity
# - Thread states
```

## 9. Files.mismatch() (Java 12+)

```java
// Find first mismatched position between two files
long mismatch = Files.mismatch(path1, path2);
if (mismatch == -1) {
    System.out.println("Files are identical");
} else {
    System.out.println("First mismatch at byte: " + mismatch);
}
```

---

## Java 11 Coding Questions

### Q1: HttpClient GET Request
Write a program using the new HttpClient API to fetch data from a URL.

```java
public String fetchData(String url) throws Exception {
    HttpClient client = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(url))
        .GET()
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    if (response.statusCode() == 200) {
        return response.body();
    }
    throw new RuntimeException("Failed: HTTP " + response.statusCode());
}
```

### Q2: Async File Processor
Read multiple files asynchronously and combine results.

```java
public CompletableFuture<String> processFiles(List<Path> paths) {
    return CompletableFuture.allOf(paths.stream()
            .map(path -> CompletableFuture.supplyAsync(() -> {
                try { return Files.readString(path); }
                catch (IOException e) { throw new RuntimeException(e); }
            }))
            .toArray(CompletableFuture[]::new))
        .thenApply(v -> paths.stream()
            .map(path -> {
                try { return Files.readString(path); }
                catch (IOException e) { return ""; }
            })
            .collect(Collectors.joining("\n")));
}
```

### Q3: String Operations Suite
Implement utility methods using new String methods.

```java
public class StringUtils {
    public static boolean isNullOrBlank(String s) {
        return s == null || s.isBlank();
    }

    public static String padRight(String s, int length, char padChar) {
        return s + String.valueOf(padChar).repeat(Math.max(0, length - s.length()));
    }

    public static List<String> splitLines(String multiline) {
        return multiline.lines().collect(Collectors.toList());
    }

    public static String stripUnicode(String s) {
        return s.strip(); // Unicode-aware whitespace removal
    }
}
```

### Q4: URL Content Downloader with Timeout
Download URL content with connect and read timeouts.

```java
public class UrlDownloader {
    private final HttpClient client;

    public UrlDownloader() {
        this.client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
    }

    public String download(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url))
            .timeout(Duration.ofSeconds(30))
            .GET()
            .build();

        return client.send(request, HttpResponse.BodyHandlers.ofString()).body();
    }

    public CompletableFuture<String> downloadAsync(String url) {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url))
            .timeout(Duration.ofSeconds(30))
            .GET()
            .build();

        return client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body);
    }
}
```

### Q5: File Comparison Tool
Compare two files and report differences.

```java
public class FileComparator {
    public static String compareFiles(Path file1, Path file2) throws IOException {
        if (Files.mismatch(file1, file2) == -1) {
            return "Files are identical";
        }

        String content1 = Files.readString(file1);
        String content2 = Files.readString(file2);

        List<String> lines1 = content1.lines().collect(Collectors.toList());
        List<String> lines2 = content2.lines().collect(Collectors.toList());

        StringBuilder diff = new StringBuilder();
        int maxLines = Math.max(lines1.size(), lines2.size());
        for (int i = 0; i < maxLines; i++) {
            if (i >= lines1.size()) {
                diff.append("+ ").append(lines2.get(i)).append("\n");
            } else if (i >= lines2.size()) {
                diff.append("- ").append(lines1.get(i)).append("\n");
            } else if (!lines1.get(i).equals(lines2.get(i))) {
                diff.append("- ").append(lines1.get(i)).append("\n");
                diff.append("+ ").append(lines2.get(i)).append("\n");
            }
        }
        return diff.toString();
    }
}
```

### Q6: Multi-File Single-Source Runner
Create a Java application that runs from multiple source files without compilation (Java 11+).

```bash
# Main.java
public class Main {
    public static void main(String[] args) {
        System.out.println(Helper.greet("World"));
    }
}

# Helper.java (in same directory)
public class Helper {
    public static String greet(String name) {
        return "Hello, " + name + "!";
    }
}

# Run: java --source 11 Main.java
```

### Q7: HTTP Request with Retry
Implement an HTTP client with exponential backoff retry using CompletableFuture.

```java
public class ResilientHttpClient {
    private final HttpClient client;
    private static final int MAX_RETRIES = 3;

    public ResilientHttpClient() {
        this.client = HttpClient.newHttpClient();
    }

    public String getWithRetry(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url)).GET().build();

        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() < 500) {
                return response.body();
            }
            if (attempt < MAX_RETRIES - 1) {
                Thread.sleep((long) Math.pow(2, attempt) * 1000);
            }
        }
        throw new RuntimeException("Max retries exceeded for: " + url);
    }

    public CompletableFuture<String> getAsyncWithRetry(String url) {
        return CompletableFuture.supplyAsync(() -> {
            try { return getWithRetry(url); }
            catch (Exception e) { throw new RuntimeException(e); }
        });
    }
}
```
