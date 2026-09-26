# Distributed Transactions - Challenges

## Why Distributed Transactions Are Hard
- Multiple services involved
- Network partitions possible
- Different databases
- Timing issues
- Partial failures

## Solutions

### 1. **Saga Pattern**
```java
// Orchestrated Saga
@Saga
public class OrderSaga {
    
    @StartSaga
    public SagaState createOrder(Order order) {
        // Step 1: Create order
        orderService.create(order);
        return SagaState.builder().orderId(order.getId()).build();
    }
    
    @SagaStep(compensation = "cancelPayment")
    public SagaState processPayment(SagaState state) {
        // Step 2: Process payment
        paymentService.charge(state.getOrderId(), state.getAmount());
        return state;
    }
    
    @SagaStep(compensation = "releaseInventory")
    public SagaState shipOrder(SagaState state) {
        // Step 3: Ship order
        shippingService.createShipment(state.getOrderId());
        return state;
    }
    
    // Compensation methods
    public void cancelPayment(SagaState state) {
        paymentService.refund(state.getOrderId());
    }
    
    public void releaseInventory(SagaState state) {
        inventoryService.release(state.getOrderId());
    }
}
```

### 2. **Two-Phase Commit (2PC)**
```java
// Coordinator-based
@TwoPhaseCommit
public interface OrderService {
    
    @Prepare
    boolean prepareOrder(Order order);
    
    @Commit
    void commitOrder(Order order);
    
    @Rollback
    void rollbackOrder(Order order);
}
```

### 3. **Event-Driven Approach**
```java
// Event sourcing with compensation
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        // Create order
        orderRepository.save(order);
        
        // Publish event
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
    }
}

@Component
public class OrderEventHandler {
    
    @Async
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Process payment
            paymentService.charge(event.getOrder());
            
            // Publish success event
            eventPublisher.publishEvent(new PaymentCompletedEvent(event.getOrder()));
        } catch (Exception ex) {
            // Publish failure event
            eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrder(), ex));
        }
    }
}
```

## Interview Tip
Explain that distributed transactions are challenging due to CAP theorem. Saga pattern is often preferred over 2PC for better availability.