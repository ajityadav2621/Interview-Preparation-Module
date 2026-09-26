# Choosing Timeout Values

## Factors to Consider

### 1. **Service Characteristics**
- **Network latency**: Geographic distribution
- **Processing time**: CPU vs I/O operations
- **Load patterns**: Peak vs average usage
- **Dependency performance**: Downstream services

### 2. **Timeout Hierarchy**
```
Client Request Timeout: 30 seconds
  ↓
API Gateway Timeout: 25 seconds
  ↓
Service Timeout: 20 seconds
  ↓
Database Timeout: 15 seconds
  ↓
External API Timeout: 10 seconds
```

## Calculation Strategies

### 1. **P99 Latency Based**
```java
// Use actual observed latency
double p99Latency = getObservedP99Latency(); // e.g., 200ms

// Set timeout at 2-3x P99
int timeout = (int) (p99Latency * 2.5); // 500ms
```

### 2. **Resource-Based**
```java
// Consider thread pool capacity
int threadPoolSize = 50;
double requestsPerSecond = 1000;

// Each request should not block threads for too long
int maxTimeout = threadPoolSize / (requestsPerSecond / 1000); // ~50ms
```

### 3. **Experience-Based**
```java
// Common defaults
HttpClient:
- Connect timeout: 1-3 seconds
- Read timeout: 5-10 seconds

Database:
- Query timeout: 10-30 seconds
- Connection timeout: 1-3 seconds

External APIs:
- Connect timeout: 1-3 seconds
- Read timeout: 5-15 seconds
```

## Best Practices

### 1. **Configure at Multiple Levels**
```java
// HTTP client level
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(3))
    .build();

// Request level
@Timeout(duration = 5, unit = TimeUnit.SECONDS)
public String callService() { ... }

// Global fallback
@CircuitBreaker(name = "service", fallbackMethod = "fallback")
```

### 2. **Monitor and Adjust**
- Track actual vs configured timeouts
- Adjust based on observed patterns
- Use P99/P95 metrics

## Interview Tip
Explain that timeouts should be based on actual performance data, not guesses. Start conservative and adjust based on monitoring.