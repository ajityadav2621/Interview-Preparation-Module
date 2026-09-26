# Database Unavailable - What Happens?

## Immediate Impact
- All database operations fail
- Application cannot function
- User-facing errors
- Potential data inconsistency

## What Should Happen

### 1. **Circuit Breaker**
```java
@CircuitBreaker(name = "database", fallbackMethod = "dbFallback")
public User getUser(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException());
}

public User dbFallback(Long id, Exception ex) {
    // Return cached user or default
    return cache.getOrDefault(id, User.getDefault());
}
```

### 2. **Cache-Aside Pattern**
```java
public User getUser(Long id) {
    // Try cache first
    User user = cache.getIfPresent(id);
    
    if (user == null) {
        try {
            // Try database
            user = userRepository.findById(id)
                .orElseThrow();
            cache.put(id, user);
        } catch (DataAccessException ex) {
            // Database unavailable
            log.warn("Database down, serving from cache");
            // Return null or throw appropriate exception
        }
    }
    
    return user;
}
```

### 3. **Graceful Degradation**
- Serve static content
- Use cached data
- Disable non-critical features
- Show appropriate messages

## Interview Tip
Explain that database failures are critical. The goal is to maintain partial functionality using caches and fallbacks.