# Backward Compatibility Between Services

## Strategies

### 1. **Versioning**
```java
// API versioning
@RestController
@RequestMapping("/api/v1")
public class UserV1Controller {
    // Old API
}

@RestController
@RequestMapping("/api/v2")
public class UserV2Controller {
    // New API
}
```

### 2. **Feature Flags**
```java
@FeatureFlag("new_user_api")
@GetMapping("/users")
public ResponseEntity<User> getUser() {
    if (featureFlag.isEnabled("new_user_api")) {
        return newUserResponse();
    } else {
        return oldUserResponse();
    }
}
```

### 3. **Schema Evolution**
```java
// Avro with backward compatibility
@AvroSchema("user-v1.avsc")
class UserV1 {
    String name;
    String email;
}

@AvroSchema("user-v2.avsc") 
class UserV2 {
    String name;
    String email;
    String phone; // New field with default
}
```

### 4. **Adapter Pattern**
```java
// Convert between versions
public UserV2 adapt(UserV1 v1) {
    return UserV2.builder()
        .name(v1.getName())
        .email(v1.getEmail())
        .phone("") // Default for new field
        .build();
}
```

## Interview Tip
Explain that backward compatibility requires careful planning. Use versioning, feature flags, and schema evolution strategies.