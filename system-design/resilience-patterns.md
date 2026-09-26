# Resilience Patterns: Downstream Failures, Cascading Failures, and Slow Services

## 1. What Happens When a Downstream Service Becomes Unavailable?

When a downstream service is unavailable, the calling service will experience:

- **Connection timeouts** — TCP connection attempts fail
- **HTTP 503/504 errors** — service returns error responses
- **Thread exhaustion** — request threads block waiting for responses
- **Cascading failures** — if not handled, the calling service may also become unavailable

### Example:
```java
// Without protection — thread exhaustion
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // If UserService is down, this blocks for the full timeout
        User user = userServiceClient.getUser(order.getUserId());
        return new Order(order, user);
    }
}
```

### Solution: Circuit Breaker + Timeout
```java
@Service
public class OrderService {
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("user-service");
    
    public Order getOrder(String id) {
        return circuitBreaker.executeSupplier(() -> {
            User user = userServiceClient.getUser(id);  // Fails fast if circuit is open
            return new Order(order, user);
        });
    }
}
```

## 2. How Would You Prevent Cascading Failures?

Cascading failures occur when a failure in one service causes failures in dependent services.

### Strategies:

1. **Circuit Breaker** — Stop sending requests to a failing service
2. **Bulkhead** — Isolate resources per service
3. **Timeout** — Fail fast instead of waiting
4. **Rate Limiting** — Prevent overwhelming downstream services
5. **Retry with Backoff** — Don't retry immediately

```java
// Bulkhead + Circuit Breaker + Timeout
public class ResilientService {
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("downstream");
    private final ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("downstream",
        ThreadPoolBulkheadConfig.custom()
            .coreThreadPoolSize(10)
            .maxThreadPoolSize(20)
            .queueCapacity(50)
            .build());
    
    public CompletableFuture<Result> callDownstream(Request request) {
        return circuitBreaker.executeCompletionStage(() ->
            bulkhead.executeCompletionStage(() ->
                externalClient.call(request)
            ).toCompletableFuture()
        ).toCompletableFuture()
        .orTimeout(5, TimeUnit.SECONDS);
    }
}
```

## 3. What Happens When a Service Becomes Extremely Slow?

A slow service causes:
- **Thread pool exhaustion** — threads are blocked waiting
- **Increased latency** — requests queue up
- **Resource waste** — connections, memory held longer
- **Cascading timeouts** — downstream services also timeout

### Solution: Timeout + Bulkhead
```java
@Bulkhead(name = "slow-service", type = Type.THREADPOOL)
@TimeLimiter(name = "slow-service")
@CircuitBreaker(name = "slow-service", fallbackMethod = "fallback")
public CompletableFuture<Result> callSlowService(Request request) {
    return CompletableFuture.supplyAsync(() -> 
        slowServiceClient.call(request));
}

public CompletableFuture<Result> fallback(Request request, Exception ex) {
    return CompletableFuture.completedFuture(Result.defaultValue());
}
```

## 4. How Do You Choose Appropriate Timeout Values?

### Guidelines:
1. **Measure actual latency** — Use percentiles (p95, p99)
2. **Add buffer** — Set timeout to 2-3x the p99 latency
3. **Consider downstream timeouts** — Your timeout should be less than downstream timeouts
4. **Use different timeouts** — Connect timeout vs read timeout

```yaml
# Example: If p99 is 200ms, set timeout to 500ms
resilience4j:
  timelimiter:
    instances:
      downstream-api:
        timeout-duration: 500ms
```

### Timeout Hierarchy:
```
Client timeout (1000ms)
  → API Gateway timeout (800ms)
    → Service timeout (500ms)
      → Database query timeout (300ms)
```

## 5. Retry vs Circuit Breaker – When Should You Use Each?

| Scenario | Retry | Circuit Breaker |
|----------|-------|-----------------|
| Transient errors (network blip) | ✅ Yes | ❌ No |
| Service is down | ❌ No | ✅ Yes |
| Rate limited | ✅ Yes (with backoff) | ❌ No |
| Service is slow | ❌ No | ✅ Yes |
| Temporary overload | ✅ Yes (with backoff) | ✅ Yes |

### Combined Approach:
```java
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2))
@CircuitBreaker(name = "api", fallbackMethod = "fallback")
public Result callApi(Request request) {
    return apiClient.call(request);
}
```

## 6. How Can Retries Make an Outage Worse?

Retries can amplify failures through:

1. **Retry storms** — Multiple clients retrying simultaneously
2. **Thundering herd** — All retries hit at the same time
3. **Increased load** — 3x retries = 3x load on the failing service

### Example:
```
1000 clients → 1000 requests → Service fails
1000 clients retry → 2000 requests → Service fails worse
1000 clients retry again → 3000 requests → Complete outage
```

### Solution: Exponential Backoff + Jitter
```java
@Retryable(
    maxAttempts = 3,
    backoff = @Backoff(
        delay = 1000,
        multiplier = 2,
        randomizationFactor = 0.5  // Jitter
    )
)
```

## 7. What Is a Bulkhead Pattern?

The Bulkhead pattern isolates resources to prevent failures from propagating.

### Types:
1. **Thread Pool Bulkhead** — Separate thread pools per service
2. **Semaphore Bulkhead** — Limit concurrent calls

```java
// Thread Pool Bulkhead
ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("payment-service",
    ThreadPoolBulkheadConfig.custom()
        .coreThreadPoolSize(5)
        .maxThreadPoolSize(10)
        .queueCapacity(20)
        .build());

// Semaphore Bulkhead
Bulkhead bulkhead = Bulkhead.of("inventory-service",
    BulkheadConfig.custom()
        .maxConcurrentCalls(10)
        .maxWaitDuration(Duration.ofMillis(100))
        .build());
```

## 8. How Do You Handle Partial Failures?

When some services succeed and others fail:

```java
public OrderResult processOrder(OrderRequest request) {
    // Step 1: Reserve inventory (may fail)
    ReservationResult reservation = inventoryService.reserve(request.getItems());
    
    // Step 2: Process payment (may fail)
    PaymentResult payment = paymentService.charge(request.getPayment());
    
    // Step 3: Confirm order (may fail)
    Order order = orderService.create(request, payment.getId());
    
    // If any step fails, compensate previous steps
    if (reservation.isSuccess() && !payment.isSuccess()) {
        inventoryService.release(reservation.getId());  // Compensate
    }
    
    return new OrderResult(order, reservation, payment);
}
```

### Better: Saga Pattern
```java
public class OrderSaga {
    public void execute(OrderRequest request) {
        String reservationId = reserveInventory(request);
        try {
            String paymentId = processPayment(request);
            try {
                createOrder(request, paymentId);
            } catch (Exception e) {
                refundPayment(paymentId);
                throw e;
            }
        } catch (Exception e) {
            releaseInventory(reservationId);
            throw e;
        }
    }
}

---

## Production Topic: Circuit Breaker (Resilience4J)

### The Problem: Service B Slows → Service A Waits → Everything Dies

```java
// Without circuit breaker — cascading failure
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // If PaymentService is slow (5 seconds),
        // this thread is blocked for 5 seconds
        // 1000 requests → 1000 threads blocked → thread pool exhausted
        // → entire application becomes unresponsive
        Payment payment = paymentServiceClient.getPayment(id);
        return new Order(order, payment);
    }
}
```

### Circuit Breaker States: CLOSED → OPEN → HALF-OPEN

```
CLOSED (normal):
  - Requests flow through to downstream service
  - Circuit breaker counts failures
  - If failure rate > threshold → OPEN

OPEN (failing fast):
  - All requests fail immediately (no call to downstream)
  - After wait duration → HALF-OPEN

HALF-OPEN (testing recovery):
  - Limited requests allowed through
  - If success → CLOSED
  - If failure → OPEN again
```

### Resilience4J Configuration

```java
// Circuit Breaker configuration
@Bean
public CircuitBreakerConfig circuitBreakerConfig() {
    return CircuitBreakerConfig.custom()
        .failureRateThreshold(50)  // Open if >50% failures
        .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before half-open
        .slidingWindowSize(10)  // Count last 10 requests
        .permittedNumberOfCallsInHalfOpenState(5)  // Allow 5 test calls
        .build();
}

// Time Limiter configuration
@Bean
public TimeLimiterConfig timeLimiterConfig() {
    return TimeLimiterConfig.custom()
        .timeoutDuration(Duration.ofSeconds(3))  // Timeout after 3s
        .build();
}

// Retry configuration
@Bean
public RetryConfig retryConfig() {
    return RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(500))
        .retryExceptions(IOException.class, TimeoutException.class)
        .build();
}
```

### Using Circuit Breaker in Code

```java
@Service
public class OrderService {
    
    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "payment-service")
    @Retry(name = "payment-service")
    public CompletableFuture<Payment> getPayment(String orderId) {
        return CompletableFuture.supplyAsync(() -> 
            paymentClient.getPayment(orderId)
        );
    }
    
    // Fallback method — same signature + Exception parameter
    public CompletableFuture<Payment> paymentFallback(String orderId, Exception e) {
        log.warn("Payment service unavailable for order: {}", orderId, e);
        return CompletableFuture.completedFuture(
            Payment.pending("Payment service unavailable")
        );
    }
}
```

### When to Use Circuit Breaker

| Scenario | Use Circuit Breaker? |
|----------|---------------------|
| Service is down | ✅ Yes — fail fast |
| Service is slow | ✅ Yes — prevent thread exhaustion |
| Transient network error | ❌ No — use retry instead |
| Rate limited | ❌ No — use retry with backoff |
| Service returns 5xx | ✅ Yes — count as failure |

### Combined Approach: Retry + Circuit Breaker

```java
// Retry for transient errors
// Circuit Breaker for persistent failures
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2))
@CircuitBreaker(name = "api", fallbackMethod = "fallback")
public Result callApi(Request request) {
    return apiClient.call(request);
}

public Result fallback(Request request, Exception e) {
    // Return cached/default value
    return Result.defaultValue();
}
```

### Key Takeaways

- **Circuit Breaker fails fast** — doesn't wait for slow services
- **Recovers gracefully** — automatically tests recovery with half-open state
- **CLOSED → OPEN → HALF-OPEN** — three states manage the lifecycle
- **Most teams add this after the incident** — not before. Be proactive.
- **Combine with Retry** — retry for transient errors, circuit breaker for persistent failures
