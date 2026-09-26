# Optimizing Slow SQL Queries

## Analysis Steps

### 1. **EXPLAIN ANALYZE**
```sql
-- See query execution plan
EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE user_id = 123 AND status = 'PENDING' 
ORDER BY created_date DESC 
LIMIT 10;

-- Key metrics:
-- Execution time
-- Rows examined vs returned
-- Index usage
-- Sort operations
```

### 2. **Index Analysis**
```sql
-- Check existing indexes
SHOW INDEX FROM orders;

-- Find missing indexes
SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

## Common Optimization Techniques

### 1. **Add Missing Indexes**
```sql
-- Before: full table scan
SELECT * FROM orders WHERE user_id = 123;

-- After: index scan
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### 2. **Optimize Query Structure**
```sql
-- Bad: SELECT * fetches all columns
SELECT * FROM users WHERE id = 123;

-- Good: Select only needed columns
SELECT id, name, email FROM users WHERE id = 123;
```

### 3. **Avoid N+1 Queries**
```java
// Bad: N+1 problem
for (Long orderId : orderIds) {
    Order order = orderRepository.findById(orderId);
    // Process order
}

// Good: Single query with JOIN
List<Order> orders = orderRepository.findByIdIn(orderIds);
```

### 4. **Use Proper JOINs**
```sql
-- Instead of correlated subquery
SELECT o.*, u.name 
FROM orders o 
WHERE o.user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- Use explicit JOIN
SELECT o.*, u.name 
FROM orders o 
INNER JOIN users u ON o.user_id = u.id 
WHERE u.status = 'ACTIVE';
```

## Query Optimization Checklist
- [ ] Use EXPLAIN ANALYZE
- [ ] Check index usage
- [ ] Avoid SELECT *
- [ ] Limit result sets
- [ ] Use pagination
- [ ] Batch operations
- [ ] Monitor slow query logs

## Interview Tip
Always start with EXPLAIN ANALYZE to understand the current execution plan. Mention that optimization should be based on actual query patterns, not assumptions.