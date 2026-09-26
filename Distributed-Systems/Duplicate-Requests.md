# Same Request Processed Twice - Handling

## Causes
- Network timeouts and retries
- User double-clicks
- Load balancer retries
- Message queue redelivery

## Solutions

### 1. **Idempotency Keys**
```java
// Unique key per request
String idempotencyKey = UUID.randomUUID().toString();

// Store processed keys
Set<String> processedKeys = ConcurrentHashMap.newKeySet();

public void process(String idempotencyKey, Request request) {
    if (!processedKeys.add(idempotencyKey)) {
        // Already processed, skip
        return;
    }
    
    // Process request
    doProcess(request);
}
```

### 2. **Database Unique Constraints**
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    client_order_id VARCHAR(255) UNIQUE NOT NULL,
    amount DECIMAL(10,2) NOT NULL
);
```

### 3. **State Machine**
```java
enum OrderStatus { PENDING, PROCESSING, COMPLETED, FAILED }

public void processOrder(Order order) {
    if (order.getStatus() != OrderStatus.PENDING) {
        return; // Already processed or processing
    }
    
    order.setStatus(OrderStatus.PROCESSING);
    // Process order
    order.setStatus(OrderStatus.COMPLETED);
}
```

## Interview Tip
Explain that duplicate processing is a common problem in distributed systems. Idempotency must be designed into the system from the start.