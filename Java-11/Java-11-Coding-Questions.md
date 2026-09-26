# Java 11 – Coding Questions

## Q1: String Utility Class
Build a utility class using Java 11 String methods: isBlank, strip, repeat, lines.

```java
public final class StringUtils {

    public static boolean isBlankOrNull(String s) {
        return s == null || s.isBlank();
    }

    public static String repeat(String s, int count) {
        return s.repeat(count);
    }

    public static String stripAndCapitalize(String s) {
        if (s == null) return null;
        String stripped = s.strip();
        if (stripped.isEmpty()) return stripped;
        return stripped.substring(0, 1).toUpperCase() + stripped.substring(1);
    }

    public static int countLines(String multiline) {
        return (int) multiline.lines().count();
    }

    public static List<String> toLineList(String multiline) {
        return multiline.lines().collect(Collectors.toList());
    }
}
```

## Q2: HTTP Response Logger
Log HTTP response details using HttpClient and BodyHandlers.

```java
public void logResponse(String url) throws Exception {
    HttpClient client = HttpClient.newHttpClient();
    HttpRequest request = HttpRequest.newBuilder(URI.create(url))
        .timeout(Duration.ofSeconds(10))
        .GET()
        .build();

    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

    System.out.println("Status: " + response.statusCode());
    System.out.println("Headers: " + response.headers().map());
    System.out.println("Body: " + response.body());
    response.headers().firstValue("Content-Type")
        .ifPresent(ct -> System.out.println("Content-Type: " + ct));
}
```

## Q3: Multi-threaded File Downloader
Download files from URLs in parallel and save to disk.

```java
public class ParallelDownloader {
    private final ExecutorService executor = Executors.newFixedThreadPool(5);

    public Map<String, Path> downloadAll(Map<String, Path> urlToPath) throws Exception {
        List<Future<Map.Entry<String, Path>>> futures = urlToPath.entrySet().stream()
            .map(entry -> executor.submit(() -> {
                HttpClient client = HttpClient.newHttpClient();
                HttpRequest request = HttpRequest.newBuilder(URI.create(entry.getKey()))
                    .GET().build();
                HttpResponse<Path> response = client.send(request, 
                    HttpResponse.BodyHandlers.writeToFile(entry.getValue()));
                return Map.entry(entry.getKey(), response.body());
            }))
            .collect(Collectors.toList());

        Map<String, Path> results = new LinkedHashMap<>();
        for (Future<Map.Entry<String, Path>> f : futures) {
            Map.Entry<String, Path> e = f.get();
            results.put(e.getKey(), e.getValue());
        }
        return results;
    }
}
```

## Q4: Text File Analyzer
Analyze a text file: line count, word count, character count, blank line count.

```java
public Map<String, Long> analyzeFile(Path path) throws IOException {
    String content = Files.readString(path);

    Map<String, Long> stats = new LinkedHashMap<>();
    stats.put("lines", content.lines().count());
    stats.put("nonBlankLines", content.lines().filter(s -> !s.isBlank()).count());
    stats.put("words", content.lines()
        .flatMapToInt(s -> s.chars())
        .filter(c -> Character.isWhitespace(c))
        .count());
    stats.put("characters", (long) content.length());
    stats.put("bytes", (long) content.getBytes(StandardCharsets.UTF_8).length);

    return stats;
}
```

## Q5: Single-File Application
Create a complete application in a single .java file with multiple classes (Java 11+).

```java
// File: LibraryApp.java
import java.util.*;

class Book {
    final String title; final String author; final int year;
    Book(String t, String a, int y) { title=t; author=a; year=y; }
    @Override public String toString() { return title + " by " + author + " (" + year + ")"; }
}

class Library {
    private final List<Book> books = new ArrayList<>();
    void add(Book b) { books.add(b); }
    List<Book> searchByAuthor(String author) {
        return books.stream().filter(b -> b.author.equalsIgnoreCase(author)).collect(Collectors.toList());
    }
    List<Book> searchByYearRange(int from, int to) {
        return books.stream().filter(b -> b.year >= from && b.year <= to).collect(Collectors.toList());
    }
}

public class LibraryApp {
    public static void main(String[] args) {
        Library lib = new Library();
        lib.add(new Book("1984", "George Orwell", 1949));
        lib.add(new Book("Animal Farm", "George Orwell", 1945));
        lib.add(new Book("Brave New World", "Aldous Huxley", 1932));

        System.out.println("Orwell books: " + lib.searchByAuthor("Orwell"));
        System.out.println("1900-1950: " + lib.searchByYearRange(1900, 1950));
    }
}
// Run: java LibraryApp.java
```
