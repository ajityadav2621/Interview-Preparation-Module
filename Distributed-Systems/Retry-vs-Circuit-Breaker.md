# Retry vs Circuit Breaker

## Retry Pattern

### When to Use
- **Transient failures**: Network glitches, temporary timeouts
- **Idempotent operations**: Safe to retry multiple times
- **Short-lived issues**: Quick recovery expected

### Implementation
```java
@Retryable(value = {Exception.class},
           maxAttempts = 3,
           backoff = @Backoff(delay = 2000))
public String callService() {
    return service.getData();
}
```

### Benefits
- Handles temporary failures
- Simple to implement
- Increases success rate

### Risks
- Can overwhelm failing service
- May cause duplicate processing
- Not suitable for non-idempotent operations

## Circuit Breaker Pattern

### When to Use
- **Persistent failures**: Service is down or degraded
- **Resource-intensive operations**: Don't want to waste resources
- ** cascading failure prevention**: Protect upstream

### Implementation
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");

Supplier<String> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> service.call());
```

### States
- **CLOSED**: Normal operation, requests pass through
- **OPEN**: Failures exceed threshold, fast fail
- **HALF-OPEN**: Test if service recovered

### Benefits
- Prevents cascading failures
- Fast failure response
- Gives service time to recover

### Risks
- May reject valid requests
- Requires proper fallback
- State management complexity

## Combined Approach
```java
@CircuitBreaker(name = "service", fallbackMethod = "fallback")
@Retryable(value = {Exception.class}, maxAttempts = 3)
public String callService() {
    return service.getData();
}
```

## Interview Tip
Explain that retries work for transient failures, while circuit breakers protect against persistent failures. Often used together for comprehensive resilience.