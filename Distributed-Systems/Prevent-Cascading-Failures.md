# Preventing Cascading Failures

## Strategies

### 1. **Circuit Breaker**
```java
// States: CLOSED → OPEN → HALF-OPEN
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50.0f)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(5)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("service", config);
```

### 2. **Bulkhead Pattern**
```java
// Isolate resources
Bulkhead bulkhead = Bulkhead.of(10, TimeUnit.SECONDS);

Supplier<String> decorated = Bulkhead
    .decorateSupplier(bulkhead, () -> externalApi.call());
```

### 3. **Timeouts and Retries**
```java
@Retryable(value = {Exception.class}, 
           maxAttempts = 3,
           backoff = @Backoff(delay = 1000))
@Timeout(duration = 5, unit = TimeUnit.SECONDS)
public String callService() {
    return service.call();
}
```

### 4. **Rate Limiting**
```java
// Per-service rate limiting
Bucket bucket = Bucket.builder()
    .addLimit(Bandwidth.classic(1000, Duration.ofMinutes(1)))
    .build();

if (bucket.tryConsume(1)) {
    // Process request
} else {
    // Reject or queue
}
```

## Interview Tip
Explain that cascading failures happen when one service failure causes others to fail. Prevention requires multiple layers of protection.