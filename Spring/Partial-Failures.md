# Handling Partial Failures with Multiple Downstream Services

## Strategies

### 1. **Circuit Breaker Pattern**
```java
// Resilience4j implementation
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("serviceB");

Supplier<String> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> serviceB.call());

String result = decorated.get();
```

### 2. **Fallback Mechanisms**
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public Payment processPayment(Order order) {
    return paymentService.charge(order);
}

public Payment fallback(Order order, Exception ex) {
    // Return cached value or default
    return new Payment(order.getId(), "PENDING");
}
```

### 3. **Bulkhead Pattern**
```java
// Isolate resources for different services
Bulkhead bulkhead = Bulkhead.of(10, TimeUnit.SECONDS);

Supplier<String> decorated = Bulkhead
    .decorateSupplier(bulkhead, () -> externalApi.call());
```

### 4. **Timeout Configuration**
```java
// Configure timeouts at multiple levels
@Retry(name = "serviceB", maxAttempts = 3)
@Timeout(duration = 2, unit = TimeUnit.SECONDS)
public String callServiceB() {
    return serviceB.getData();
}
```

## Interview Tip
Explain that partial failures are inevitable in distributed systems. The goal is to fail gracefully and maintain partial functionality.