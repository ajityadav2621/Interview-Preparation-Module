# Choreography vs Orchestration

## Choreography Pattern
- Services communicate through events
- No central coordinator
- Loose coupling
- Complex event flows

### Example
```java
// Service A publishes event
eventPublisher.publishEvent(new OrderCreatedEvent(order));

// Service B listens and responds
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) {
    paymentService.charge(event.getOrder());
}

// Service C listens to payment result
@EventListener  
public void handlePaymentCompleted(PaymentCompletedEvent event) {
    shippingService.createShipment(event.getOrder());
}
```

## Orchestration Pattern
- Central coordinator manages flow
- Explicit steps and compensations
- Easier to understand and debug
- Tighter coupling

### Example
```java
@Saga
public class OrderOrchestrator {
    
    @StartSaga
    public SagaState createOrder(Order order) {
        return SagaState.builder().orderId(order.getId()).build();
    }
    
    @SagaStep(compensation = "cancelOrder")
    public SagaState processPayment(SagaState state) {
        paymentService.charge(state.getOrderId(), state.getAmount());
        return state;
    }
    
    @SagaStep(compensation = "releaseInventory")
    public SagaState shipOrder(SagaState state) {
        shippingService.createShipment(state.getOrderId());
        return state;
    }
}
```

## When to Use Each

### Use Choreography When:
- **Loose coupling** desired
- **Complex event interactions**
- **Many services involved**
- **Event-driven architecture**

### Use Orchestration When:
- **Clear business process** exists
- **Easy to understand flow**
- **Centralized control needed**
- **Simple compensation logic**

## Interview Tip
Explain that choreography offers loose coupling but complex flows, while orchestration provides clarity at the cost of tight coupling. Choose based on team size and complexity.