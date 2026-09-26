# Saga Step Fails - What Happens?

## Compensation Flow

### 1. **Normal Flow**
```
Step 1: Create Order → Success
Step 2: Process Payment → Success  
Step 3: Ship Order → Success
```

### 2. **Failure Flow**
```
Step 1: Create Order → Success
Step 2: Process Payment → Failed
Step 3: Compensate Step 1 (Cancel Order)
```

## Implementation

### 1. **Orchestrated Saga**
```java
@Saga
public class OrderSaga {
    
    @StartSaga
    public SagaState createOrder(Order order) {
        orderService.create(order);
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
    
    // Compensation methods
    public void cancelOrder(SagaState state) {
        orderService.cancel(state.getOrderId());
    }
    
    public void releaseInventory(SagaState state) {
        inventoryService.release(state.getOrderId());
    }
}
```

### 2. **Choreographed Saga**
```java
// Event-based compensation
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
    }
}

@Component
public class PaymentHandler {
    
    @Async
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            paymentService.charge(event.getOrder());
            eventPublisher.publishEvent(new PaymentCompletedEvent(event.getOrder()));
        } catch (Exception ex) {
            eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrder(), ex));
        }
    }
}

@Component  
public class InventoryHandler {
    
    @Async
    @EventListener
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        try {
            inventoryService.reserve(event.getOrder());
            eventPublisher.publishEvent(new InventoryReservedEvent(event.getOrder()));
        } catch (Exception ex) {
            eventPublisher.publishEvent(new InventoryFailedEvent(event.getOrder(), ex));
        }
    }
}
```

## Interview Tip
Explain that Saga pattern ensures eventual consistency. Compensation steps must be idempotent and handle failures gracefully.