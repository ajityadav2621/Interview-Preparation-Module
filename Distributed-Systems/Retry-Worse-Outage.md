# How Retries Can Make Outages Worse

## The Problem

### 1. **Thundering Herd**
- Multiple clients retry simultaneously
- Increased load on recovering service
- Can prevent recovery

### 2. **Resource Exhaustion**
- Thread pools consumed by retries
- Connection pools exhausted
- Memory pressure

### 3. **Cascading Failures**
- Retries hit already overloaded services
- Increased latency causes timeouts
- More retries → worse situation

## Example
```
Service A → Service B → Service C

Service B becomes slow:
1. Service A retries 3 times
2. Each retry waits for Service B
3. Service B gets 3x more requests
4. Service B becomes completely unavailable
5. Service A exhausts its thread pool
6. Service A cannot handle other requests
```

## Prevention

### 1. **Exponential Backoff**
```java
@Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000)
public String callService() { ... }
```

### 2. **Jitter**
```java
// Add random delay to prevent synchronization
long delay = baseDelay + random.nextInt(1000);
Thread.sleep(delay);
```

### 3. **Circuit Breaker**
```java
// Stop retrying when failures exceed threshold
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
```

## Interview Tip
Explain that retries should be used judiciously. Always combine with circuit breakers and exponential backoff to prevent making outages worse.