# One Microservice Goes Down During Checkout. What Should Happen to the Overall Workflow?

## The Problem

In a microservices architecture, a failure in one service can cascade and bring down the entire system. During checkout, if a critical service (e.g., Payment Service) goes down, the entire checkout workflow should not fail catastrophically.

## The Solution: Resilience Patterns

### 1. Circuit Breaker Pattern

The circuit breaker prevents cascading failures by temporarily stopping requests to a failing service.

```java
@Service
public class PaymentService {
    
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("payment");
    
    public PaymentResult processPayment(PaymentRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            // Call the actual payment service
            return paymentClient.process(request);
        });
    }
}
```

**States**:
- **CLOSED**: Normal operation — requests flow through
- **OPEN**: Service is failing — requests are rejected immediately
- **HALF-OPEN**: Testing if the service has recovered — limited requests allowed

```java
// Circuit breaker configuration
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)           // Open after 50% failures
    .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before retry
    .slidingWindowSize(10)              // Evaluate last 10 requests
    .build();
```

### 2. Bulkhead Pattern

Isolate failures by limiting the number of concurrent calls to a service:

```java
@Service
public class PaymentService {
    private final ThreadPoolBulkhead bulkhead = 
        ThreadPoolBulkhead.of("payment", ThreadPoolBulkheadConfig.custom()
            .coreThreadPoolSize(5)
            .maxThreadPoolSize(10)
            .build());
    
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return bulkhead.executeSupplier(() -> {
            return paymentClient.process(request);
        });
    }
}
```

### 3. Retry with Exponential Backoff

```java
@Retryable(
    value = {PaymentServiceException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.process(request);
}

@Recover
public PaymentResult fallback(PaymentServiceException ex, PaymentRequest request) {
    // Fallback logic — e.g., queue for later processing
    return PaymentResult.pending("Payment queued for retry");
}
```

### 4. Timeout Pattern

```java
public PaymentResult processPayment(PaymentRequest request) {
    return circuitBreaker.executeSupplier(
        TimeLimiter.decorateSupplier(
            timeoutExecutor,
            () -> paymentClient.process(request)
        )
    );
}
```

### 5. Fallback Pattern

```java
@Service
public class CheckoutService {
    
    public CheckoutResult checkout(CheckoutRequest request) {
        try {
            // Try to process payment
            PaymentResult payment = paymentService.processPayment(request.getPayment());
            return CheckoutResult.success(payment);
        } catch (PaymentServiceException e) {
            // Fallback: queue the order for later processing
            orderQueue.enqueue(request);
            return CheckoutResult.pending("Order queued — payment will be processed later");
        }
    }
}
```

## Complete Resilience Example

```java
@Service
public class ResilientCheckoutService {
    
    private final PaymentServiceClient paymentClient;
    private final OrderServiceClient orderClient;
    private final InventoryServiceClient inventoryClient;
    
    // Circuit breakers for each service
    private final CircuitBreaker paymentCircuitBreaker;
    private final CircuitBreaker inventoryCircuitBreaker;
    
    // Bulkheads for resource isolation
    private final ThreadPoolBulkhead paymentBulkhead;
    private final ThreadPoolBulkhead inventoryBulkhead;
    
    public CheckoutResult checkout(CheckoutRequest request) {
        // Step 1: Reserve inventory (with circuit breaker and timeout)
        ReservationResult reservation = inventoryCircuitBreaker.executeSupplier(
            TimeLimiter.decorateSupplier(
                timeoutExecutor,
                () -> inventoryBulkhead.executeSupplier(() -> 
                    inventoryClient.reserve(request.getItems())
                ).toCompletableFuture().join()
            )
        );
        
        if (!reservation.isSuccess()) {
            return CheckoutResult.failed("Inventory not available");
        }
        
        // Step 2: Process payment (with circuit breaker, retry, and fallback)
        try {
            PaymentResult payment = paymentCircuitBreaker.executeSupplier(
                TimeLimiter.decorateSupplier(
                    timeoutExecutor,
                    () -> paymentBulkhead.executeSupplier(() -> 
                        paymentClient.process(request.getPayment())
                    ).toCompletableFuture().join()
                )
            );
            
            // Step 3: Confirm order
            orderClient.confirm(request.getOrderId(), payment.getTransactionId());
            return CheckoutResult.success(payment);
            
        } catch (Exception e) {
            // Payment failed — release inventory and queue for retry
            inventoryClient.release(reservation.getReservationId());
            orderQueue.enqueue(request);
            return CheckoutResult.pending("Order queued for payment retry");
        }
    }
}
```

## Saga Pattern for Distributed Transactions

For long-running transactions across multiple services, use the Saga pattern:

```java
@Service
public class CheckoutSaga {
    
    public void execute(CheckoutRequest request) {
        // Step 1: Reserve inventory
        String reservationId = inventoryService.reserve(request.getItems());
        
        try {
            // Step 2: Process payment
            String paymentId = paymentService.charge(request.getPayment());
            
            try {
                // Step 3: Create order
                orderService.create(request, paymentId);
                
            } catch (Exception e) {
                // Compensate: refund payment
                paymentService.refund(paymentId);
                throw e;
            }
            
        } catch (Exception e) {
            // Compensate: release inventory
            inventoryService.release(reservationId);
            throw e;
        }
    }
}
```

## Event-Driven Architecture

Use events to decouple services and handle failures gracefully:

```java
// When payment service is down, publish an event
@EventListener
public void handlePaymentFailed(PaymentFailedEvent event) {
    // Queue the order for retry
    orderRetryQueue.enqueue(event.getOrderId());
    
    // Notify the user
    notificationService.send(
        event.getUserId(), 
        "Your payment is being processed. We'll notify you when complete."
    );
}

// When payment service recovers
@EventListener
public void handlePaymentServiceRecovered(ServiceRecoveredEvent event) {
    // Process queued orders
    orderRetryQueue.drainAndProcess();
}
```

## Key Strategies Summary

| Strategy | When to Use | Benefit |
|----------|-------------|---------|
| Circuit Breaker | Service is failing repeatedly | Prevents cascading failures |
| Bulkhead | Resource isolation needed | Limits blast radius |
| Retry | Transient failures | Automatic recovery |
| Timeout | Slow responses | Prevents resource exhaustion |
| Fallback | Service unavailable | Graceful degradation |
| Saga | Distributed transactions | Atomicity across services |
| Event-driven | Loose coupling needed | Resilience and scalability |

## Key Takeaway

> When a microservice goes down during checkout, the system should **degrade gracefully** rather than fail completely. Use **circuit breakers** to prevent cascading failures, **bulkheads** to isolate resources, **retries with backoff** for transient failures, and **fallbacks** to provide alternative paths. For critical operations, use the **Saga pattern** with compensating transactions, or an **event-driven architecture** to queue work for later processing.
