# Two Requests Create the Same Order at Almost the Same Time. How Do You Prevent Duplicates?

## The Problem

In a distributed system, two requests can arrive at nearly the same time and both try to create the same order. Without proper handling, this results in **duplicate orders**.

```
Time: 0ms     10ms    20ms    30ms    40ms    50ms
      |       |       |       |       |       |
      Request1 arrives
              |       |       |       |       |
              Request2 arrives (same order details)
                      |       |       |       |
                      Check: order doesn't exist
                      |       |       |       |
                      Create order
                              |       |       |
                              Check: order doesn't exist (race!)
                              |       |       |
                              Create order (DUPLICATE!)
```

## Solutions

### 1. Database-Level Unique Constraint (Primary Defense)

The most reliable approach is to enforce uniqueness at the database level:

```sql
-- Create a unique constraint on a business key
ALTER TABLE orders ADD CONSTRAINT uk_order_unique 
UNIQUE (user_id, product_id, order_date);

-- Or use a composite key
CREATE UNIQUE INDEX idx_order_dedup 
ON orders (user_id, product_id, DATE(created_at));
```

```java
// In your service:
@Transactional
public Order createOrder(OrderRequest request) {
    try {
        return orderRepository.save(new Order(request));
    } catch (DataIntegrityViolationException e) {
        // Duplicate detected by database constraint
        // Return the existing order instead
        return orderRepository.findByUserIdAndProductIdAndDate(
            request.getUserId(), request.getProductId(), request.getDate());
    }
}
```

**Pros**: Guaranteed by the database, works across all instances
**Cons**: Exception-based flow control is not ideal

### 2. Database Upsert (Recommended)

Use `INSERT ... ON CONFLICT` (PostgreSQL) or `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL):

```sql
-- PostgreSQL
INSERT INTO orders (user_id, product_id, amount, created_at)
VALUES (?, ?, ?, ?)
ON CONFLICT (user_id, product_id, DATE(created_at)) 
DO NOTHING
RETURNING *;

-- MySQL
INSERT INTO orders (user_id, product_id, amount, created_at)
VALUES (?, ?, ?, ?)
ON DUPLICATE KEY UPDATE id = LAST_INSERT_ID(id)
```

```java
public Order createOrder(OrderRequest request) {
    Order order = orderRepository.upsert(request);
    if (order == null) {
        // Another request already created this order
        order = orderRepository.findByBusinessKey(request);
    }
    return order;
}
```

### 3. Distributed Lock (Redis)

Use a distributed lock to serialize order creation:

```java
@Service
public class OrderService {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public Order createOrder(OrderRequest request) {
        String lockKey = "order_lock:" + request.getUserId() + ":" + request.getProductId();
        String lockValue = UUID.randomUUID().toString();
        
        try {
            // Acquire lock with 10-second timeout
            Boolean acquired = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10));
            
            if (!acquired) {
                // Another request is creating this order — wait and return existing
                Thread.sleep(100);
                return orderRepository.findByBusinessKey(request);
            }
            
            // Check if order already exists (double-check)
            Order existing = orderRepository.findByBusinessKey(request);
            if (existing != null) {
                return existing;
            }
            
            // Create the order
            return orderRepository.save(new Order(request));
            
        } finally {
            // Release lock
            String currentValue = redisTemplate.opsForValue().get(lockKey);
            if (lockValue.equals(currentValue)) {
                redisTemplate.delete(lockKey);
            }
        }
    }
}
```

### 4. Idempotency Key (API-Level)

Require clients to send an idempotency key:

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody OrderRequest request) {
    
    // Check if this request was already processed
    Order existing = orderRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) {
        return ResponseEntity.ok(existing);
    }
    
    // Create new order with idempotency key
    Order order = new Order(request, idempotencyKey);
    return ResponseEntity.ok(orderRepository.save(order));
}
```

### 5. Two-Phase Check (Application-Level)

```java
@Transactional
public Order createOrder(OrderRequest request) {
    // Step 1: Check if order already exists
    Order existing = orderRepository.findByBusinessKey(request);
    if (existing != null) {
        return existing;
    }
    
    // Step 2: Create the order
    return orderRepository.save(new Order(request));
}
```

**Note**: This approach has a race condition between the check and the insert. It must be combined with a database constraint for safety.

## Recommended Approach: Defense in Depth

Use **multiple layers** of protection:

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Layer 1: Idempotency key (if provided by client)
        if (request.getIdempotencyKey() != null) {
            Order existing = orderRepository.findByIdempotencyKey(request.getIdempotencyKey());
            if (existing != null) {
                return existing;
            }
        }
        
        // Layer 2: Business key check
        Order existing = orderRepository.findByBusinessKey(request);
        if (existing != null) {
            return existing;
        }
        
        // Layer 3: Database unique constraint (final safety net)
        try {
            return orderRepository.save(new Order(request));
        } catch (DataIntegrityViolationException e) {
            // Another request won the race — return the existing order
            return orderRepository.findByBusinessKey(request);
        }
    }
}
```

## Database Schema Example

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE,  -- For idempotency
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    -- Business key for deduplication
    UNIQUE (user_id, product_id, DATE(created_at))
);

CREATE INDEX idx_orders_user_product_date 
ON orders (user_id, product_id, DATE(created_at));
```

## Key Takeaway

> **Always use database-level constraints as the final safety net** for preventing duplicates. Combine this with application-level checks (idempotency keys, business key lookups) for better user experience. The database constraint guarantees correctness even in the face of race conditions, while application-level checks provide faster feedback and better error handling.
