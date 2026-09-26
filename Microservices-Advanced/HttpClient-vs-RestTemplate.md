# HttpClient vs RestTemplate

## HttpClient (Java 11+)

### Features
- **Modern**: Java's built-in HTTP client
- **Non-blocking**: Supports async operations
- **Flow-based**: Uses `CompletableFuture` for async
- **WebSocket**: Built-in support

### Example
```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

// Synchronous request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"key\":\"value\"}"))
    .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());
```

## RestTemplate (Spring)

### Features
- **Blocking**: Synchronous operations
- **Template Pattern**: Simplifies HTTP calls
- **Spring Integration**: Works with Spring ecosystem
- **Legacy**: Older but widely used

### Example
```java
// POST request
ResponseEntity<Response> response = restTemplate.postForEntity(
    "https://api.example.com/data",
    requestEntity,
    Response.class
);

// GET request
ResponseEntity<Response> response = restTemplate.getForEntity(
    "https://api.example.com/data/{id}",
    Response.class,
    id
);
```

## Key Differences

| Aspect | HttpClient | RestTemplate |
|--------|------------|--------------|
| **Introduction** | Java 11 | Spring Framework |
| **Async Support** | Yes (non-blocking) | No (blocking) |
| **API Style** | Modern, fluent | Template-based |
| **Memory Usage** | Lower (async) | Higher (blocking) |
| **Learning Curve** | Steeper | Easier for Spring users |
| **WebSocket** | Built-in | Not supported |

## When to Use Each

### Use HttpClient When:
- **Java 11+** available
- **Async operations** needed
- **High concurrency** requirements
- **Lower resource usage** desired

### Use RestTemplate When:
- **Spring Boot application**
- **Simple synchronous calls**
- **Team familiar with Spring**
- **Legacy codebase**

## Interview Tip
Explain that HttpClient is the modern replacement for RestTemplate. RestTemplate is being phased out in favor of HttpClient and WebClient.