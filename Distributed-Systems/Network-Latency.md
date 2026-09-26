# Network Latency Suddenly Increases - What Happens?

## Impact
- Slow response times
- Timeouts
- Resource exhaustion
- Cascading failures

## What Should Happen

### 1. **Adjust Timeouts**
```java
// Dynamic timeout adjustment
@Timeout(duration = calculateTimeout(), unit = TimeUnit.SECONDS)
public String callService() {
    return service.getData();
}

private int calculateTimeout() {
    // Base timeout + latency buffer
    return baseTimeout + (int) (currentLatency * 2);
}
```

### 2. **Circuit Breaker**
```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slowCallRateThreshold(50.0f)
    .slowCallDuration(Duration.ofSeconds(5))
    .build();
```

### 3. **Graceful Degradation**
```java
@CircuitBreaker(name = "service", fallbackMethod = "fallback")
public Data callService() {
    return service.call();
}

public Data fallback(Exception ex) {
    // Return cached data or default
    return cache.getOrDefault("default", Data.getDefault());
}
```

## Interview Tip
Explain that network latency is common in distributed systems. The goal is to fail fast and serve cached data when possible.