# Posting a Message to REST API Endpoint

## Using RestTemplate

### POST Request
```java
// Simple POST with JSON
ResponseEntity<String> response = restTemplate.postForEntity(
    "https://api.example.com/users",
    "{\"name\":\"John\",\"email\":\"john@example.com\"}",
    String.class
);

// POST with headers
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);

HttpEntity<String> requestEntity = new HttpEntity<>(
    "{\"name\":\"John\"}",
    headers
);

ResponseEntity<Response> response = restTemplate.postForEntity(
    "https://api.example.com/users",
    requestEntity,
    Response.class
);
```

### POST with Object
```java
User user = new User("John", "john@example.com");

// Using Request/Response objects
ResponseEntity<User> response = restTemplate.postForEntity(
    "https://api.example.com/users",
    user,
    User.class
);

// POST with URI variables
Map<String, String> uriVariables = new HashMap<>();
uriVariables.put("id", "123");

ResponseEntity<User> response = restTemplate.postForEntity(
    "https://api.example.com/users/{id}",
    user,
    User.class,
    uriVariables
);
```

## Using HttpClient (Java 11+)

### POST Request
```java
// Create object mapper
ObjectMapper objectMapper = new ObjectMapper();

// Convert object to JSON
String json = objectMapper.writeValueAsString(user);

// Build request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .build();

// Send request
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// Parse response
User responseUser = objectMapper.readValue(response.body(), User.class);
```

## Interview Tip
Explain that both approaches are valid. RestTemplate is simpler for Spring users, while HttpClient offers better performance for high-throughput applications.