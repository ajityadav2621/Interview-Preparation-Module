# Request Idempotency in Backend Services

## What is Idempotency?
An operation that can be applied multiple times without changing the result beyond the initial application.

## Why Need Idempotency?
- Network retries can cause duplicate requests
- Client timeouts may lead to retries
- Load balancers may retry requests
- User double-clicks can cause duplicate submissions

## Implementation Strategies

### 1. **Idempotency Key**
```java
Entities
- Client generates unique idempotency key
- Server stores key with request processing status
- Subsequent requests with same key return cached result

// Request
POST /payments
{
  "idempotencyKey": "req_12345",
  "amount": 100.00
}

// Server stores: idempotencyKey -> result
// Second request with same key returns same result
```

### 2. **Database Constraints**
```java
// Unique constraint on idempotency key
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    amount DECIMAL(10,2),
    status VARCHAR(50)
);
```

### 3. **State Machine Approach**
```java
enum PaymentStatus {
    PENDING, PROCESSING, COMPLETED, FAILED
}

// Only process if current status allows transition
if (payment.getStatus() == PaymentStatus.PENDING) {
    payment.setStatus(PaymentStatus.PROCESSING);
    // Process payment
}
```

## Interview Tip
Explain that idempotency is critical for financial operations. Mention that it's easier to implement at the API gateway level.