# Downstream Service Unavailable - What Happens?

## Immediate Impact
- Requests to downstream service fail
- Timeouts if not handled properly
- Cascading failures possible
- User-facing errors

## What Should Happen

### 1. **Fail Fast**
```java
// Don't wait indefinitely
@Timeout(duration = 2, unit = TimeUnit.SECONDS)
public String callDownstream() {
    return downstreamService.getData();
}
```

### 2. **Circuit Breaker Pattern**
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("downstream");

Supplier<String> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> downstreamService.call());

// When failures exceed threshold, circuit opens
// Fast fail instead of waiting
```

### 3. **Fallback Response**
```java
@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
public Payment processPayment(Order order) {
    return paymentService.charge(order);
}

public Payment fallback(Order order, Exception ex) {
    // Return cached value, default, or queue for later
    return new Payment(order.getId(), "QUEUED");
}
```

### 4. **Retry with Backoff**
```java
@Retryable(value = {ServiceUnavailableException.class},
           maxAttempts = 3,
           backoff = @Backoff(delay = 2000))
public String callService() {
    return service.call();
}
```

## Interview Tip
Explain that the goal is to fail gracefully and maintain partial functionality. Never let one service failure take down the entire system.