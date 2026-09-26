# Database Connection Pool Exhaustion

## Causes
- Too many concurrent requests
- Connection leaks
- Insufficient pool size
- Slow database queries
- Deadlocked connections

## Symptoms
- "Connection pool exhausted" errors
- High latency
- Timeout errors
- Thread blocking

## Solutions

### 1. **Increase Pool Size**
```java
// HikariCP configuration
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.connection-timeout=30000
```

### 2. **Connection Leak Detection**
```java
// Enable leak detection
spring.datasource.hikari.leak-detection-threshold=60000

// Use try-with-resources
try (Connection conn = dataSource.getConnection()) {
    // Use connection
} // Automatically closes
```

### 3. **Query Optimization**
- Add indexes for slow queries
- Use connection timeout
- Implement query caching
- Batch operations

### 4. **Connection Pool Monitoring**
```java
// Monitor pool usage
spring.datasource.hikari.metrics.enabled=true

// Key metrics:
# - Active connections
# - Idle connections  
# - Wait time
# - Connection creation rate
```

## Interview Tip
Explain that connection pool exhaustion is often caused by connection leaks. Always use try-with-resources or ensure connections are properly closed.