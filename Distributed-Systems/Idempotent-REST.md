# Making REST API Idempotent

## Strategies

### 1. **Idempotency Key**
```java
// Client-generated unique key
POST /payments
{
  "idempotencyKey": "req_12345",
  "amount": 100.00
}

// Server stores and checks
@Service
public class PaymentService {
    public Payment process(PaymentRequest request) {
        Payment existing = repository.findByIdempotencyKey(request.getIdempotencyKey());
        if (existing != null) {
            return existing; // Return existing result
        }
        
        // Process new payment
        Payment payment = new Payment(request);
        return repository.save(payment);
    }
}
```

### 2. **HTTP Methods**
- **GET**: Naturally idempotent
- **PUT**: Should be idempotent
- **DELETE**: Should be idempotent
- **POST**: Not naturally idempotent, requires design

### 3. **Database Constraints**
```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL
);
```

## Interview Tip
Explain that idempotency is critical for financial operations. POST requests are most challenging and require explicit design.