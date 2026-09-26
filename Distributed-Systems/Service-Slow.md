# Service Becomes Extremely Slow - What Happens?

## Impact
- Thread pools exhausted
- Connection pools exhausted
- Resource contention
- Timeout cascades
- Potential system failure

## What Should Happen

### 1. **Fail Fast**
```java
// Short timeout to prevent thread blocking
@Timeout(duration = 2, unit = TimeUnit.SECONDS)
public String callSlowService() {
    return slowService.getData();
}
```

### 2. **Circuit Breaker**
```java
// Open circuit after slow responses
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slowCallRateThreshold(50.0f)
    .slowCallDuration(Duration.ofSeconds(5))
    .build();
```

### 3. **Bulkhead Isolation**
```java
// Separate thread pools for different services
ExecutorService criticalExecutor = Executors.newFixedThreadPool(10);
ExecutorService nonCriticalExecutor = Executors.newFixedThreadPool(5);
```

### 4. **Fallback Response**
```java
@CircuitBreaker(name = "slowService", fallbackMethod = "slowServiceFallback")
public Data callSlowService() {
    return slowService.getData();
}

public Data slowServiceFallback(Exception ex) {
    // Return cached data or default
    return cache.getOrDefault("default", Data.getDefault());
}
```

## Interview Tip
Explain that slow services are more dangerous than completely down services because they consume resources gradually. The goal is to isolate and fail fast.