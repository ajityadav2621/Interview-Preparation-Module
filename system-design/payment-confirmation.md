# A Payment Succeeds, but Order Service Doesn't Receive the Confirmation. How Would You Handle This?

## The Problem

In a distributed system, a payment can succeed in the Payment Service, but the confirmation may never reach the Order Service due to:

1. **Network failure** — the HTTP response is lost
2. **Service crash** — Order Service is down when the callback arrives
3. **Timeout** — the callback takes too long and is dropped
4. **Message queue failure** — the confirmation message is lost

This creates an **inconsistent state**: the customer is charged, but their order is not created.

## The Solution: Reliable Event-Driven Architecture

### 1. Event Sourcing with Outbox Pattern

Instead of relying on callbacks, use an **event-driven** approach with guaranteed delivery:

```java
// Payment Service
@Service
public class PaymentService {
    
    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Process the payment
        Payment payment = paymentGateway.charge(request);
        
        // 2. Record the payment event in the same transaction
        PaymentProcessedEvent event = new PaymentProcessedEvent(
            payment.getId(),
            request.getOrderId(),
            payment.getAmount(),
            payment.getStatus()
        );
        eventRepository.save(event);  // Same DB transaction
        
        // 3. Return success
        return PaymentResult.success(payment);
    }
}
```

```java
// Separate process to publish events to Kafka
@Component
public class EventPublisher {
    
    @Scheduled(fixedDelay = 1000)  // Run every second
    public void publishEvents() {
        List<PaymentProcessedEvent> events = eventRepository
            .findUnpublishedEvents();
        
        for (PaymentProcessedEvent event : events) {
            try {
                kafkaTemplate.send("payment-processed", event);
                event.markAsPublished();  // Update in DB
            } catch (Exception e) {
                log.error("Failed to publish event", e);
                // Event remains unpublished — will retry
            }
        }
    }
}
```

### 2. Idempotent Event Handling

The Order Service must handle duplicate events gracefully:

```java
// Order Service
@Service
public class OrderEventHandler {
    
    @KafkaListener(topics = "payment-processed")
    @Transactional
    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        // Check if this event was already processed
        if (processedEventRepository.existsByEventId(event.getEventId())) {
            log.info("Event {} already processed — skipping", event.getEventId());
            return;
        }
        
        // Process the payment confirmation
        if (event.getStatus() == PaymentStatus.SUCCESS) {
            orderService.confirmOrder(event.getOrderId());
        } else {
            orderService.cancelOrder(event.getOrderId());
        }
        
        // Record that this event was processed
        processedEventRepository.save(new ProcessedEvent(event.getEventId()));
    }
}
```

### 3. Polling for Confirmation

As a fallback, the Order Service can poll the Payment Service:

```java
// Order Service — polling fallback
@Component
public class PaymentConfirmationPoller {
    
    @Scheduled(fixedDelay = 30000)  // Every 30 seconds
    public void checkPendingPayments() {
        List<Order> pendingOrders = orderRepository
            .findByStatus(OrderStatus.PENDING_PAYMENT);
        
        for (Order order : pendingOrders) {
            PaymentStatus status = paymentService.checkPaymentStatus(order.getId());
            if (status == PaymentStatus.SUCCESS) {
                orderService.confirmOrder(order.getId());
            }
        }
    }
}
```

### 4. Dead Letter Queue (DLQ)

For events that can't be processed, use a DLQ:

```java
@KafkaListener(topics = "payment-processed")
public void handlePaymentProcessed(PaymentProcessedEvent event,
                                   @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
    try {
        processEvent(event);
    } catch (Exception e) {
        // Send to DLQ for manual inspection
        kafkaTemplate.send("payment-processed-dlq", event);
        log.error("Failed to process event — sent to DLQ", e);
    }
}
```

### 5. Saga Pattern for Distributed Transactions

Use the Saga pattern to ensure consistency:

```java
@Service
public class CheckoutSaga {
    
    public void execute(CheckoutRequest request) {
        // Step 1: Reserve payment
        String paymentId = paymentService.reserve(request.getAmount());
        
        try {
            // Step 2: Create order
            Order order = orderService.create(request, paymentId);
            
            // Step 3: Confirm payment
            paymentService.confirm(paymentId);
            
        } catch (Exception e) {
            // Compensate: cancel payment reservation
            paymentService.cancel(paymentId);
            throw e;
        }
    }
}
```

## Complete Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Payment Service │    │  Event Publisher │    │  Kafka Topic    │
│                  │    │                  │    │  payment-       │
│  1. Process      │    │  3. Publish      │    │  processed      │
│     payment       │───▶│     events       │───▶│                 │
│  2. Save event   │    │                  │    │                 │
│     (same tx)    │    │  4. Mark as      │    │                 │
│                  │    │     published    │    │                 │
└─────────────────┘    └──────────────────┘    └────────┬────────┘
                                                         │
                                                         ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Order Service  │    │  Polling Fallback│    │  DLQ            │
│                 │    │                  │    │                 │
│  5. Consume     │◀───│  6. Poll for     │◀───│  Failed events  │
│     events      │    │     missing      │    │                 │
│  7. Process     │    │     payments     │    │                 │
│     (idempotent)│    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## Key Components

### 1. Outbox Table

```sql
CREATE TABLE payment_events (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(255) NOT NULL,
    payment_id VARCHAR(255) NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(50),
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_events_unpublished 
ON payment_events (published, created_at);
```

### 2. Processed Events Table

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### 3. Configuration

```yaml
# Kafka configuration
spring:
  kafka:
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false
    producer:
      retries: 3
      acks: all
```

## Key Takeaway

> When a payment succeeds but the confirmation is lost, use an **event-driven architecture** with the **Outbox Pattern** to guarantee delivery. The Payment Service records events in the same database transaction as the payment, and a separate process publishes them to Kafka. The Order Service consumes these events **idempotently** and has a **polling fallback** to catch any missed events. This ensures **eventual consistency** without losing any payment confirmations.
