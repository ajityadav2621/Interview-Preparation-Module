# API Versioning Strategies

## Common Approaches

### 1. **Path Versioning**
```java
// v1
GET /api/v1/users
GET /api/v2/users

// v2 has different response format
@RestController
@RequestMapping("/api/v2")
public class UserV2Controller {
    // New response format
}
```

### 2. **Header Versioning**
```java
// Client specifies version in header
GET /api/users
Accept: application/json;version=v2

// Server routes to appropriate version
@RestController
public class UserController {
    @GetMapping(value = "/users", headers = "X-API-Version=v2")
    public ResponseEntity<UserV2> getUserV2() { ... }
}
```

### 3. **Query Parameter Versioning**
```java
// Version specified as query parameter
GET /api/users?version=v2

// Media type versioning
Accept: application/vnd.myapi.v2+json
```

## Migration Strategies

### 1. **Parallel Support**
- Run both versions simultaneously
- Gradually migrate clients
- Monitor both versions

### 2. **Gateway Routing**
- Route requests based on version
- Apply different transformations
- Handle backward compatibility

## Interview Tip
Explain that versioning should be planned from day one. Mention that header-based versioning is often preferred for cleaner URLs.