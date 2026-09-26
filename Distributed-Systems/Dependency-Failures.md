# Microservice Surviving Dependency Failures

## Strategies

### 1. **Circuit Breaker**
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("dependency");

Supplier<String> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> dependency.call());
```

### 2. **Timeouts**
```java
@Timeout(duration = 2, unit = TimeUnit.SECONDS)
public String callDependency() {
    return dependency.getData();
}
```

### 3. **Fallback Responses**
```java
@CircuitBreaker(name = "dependency", fallbackMethod = "fallback")
public Data callDependency() {
    return dependency.call();
}

public Data fallback(Exception ex) {
    return Data.getDefault();
}
```

### 4. **Bulkhead Isolation**
```java
Bulkhead bulkhead = Bulkhead.of(10, TimeUnit.SECONDS);

Supplier<String> decorated = Bulkhead
    .decorateSupplier(bulkhead, () -> dependency.call());
```

## Interview Tip
Explain that microservices must be resilient to dependency failures. Use circuit breakers, timeouts, fallbacks, and bulkheads for protection.