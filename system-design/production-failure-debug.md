# Your Service Works in Staging but Randomly Fails in Production. Where Do You Start?

## The Problem

A service that works perfectly in staging fails randomly in production. This is a classic **environment-specific** issue that requires a systematic approach to diagnose.

## Investigation Strategy

### 1. Compare Environments

Create a checklist comparing staging and production:

| Aspect | Staging | Production | Difference |
|--------|---------|------------|------------|
| **Data volume** | Small dataset | Large dataset | Scale |
| **Traffic** | Low, predictable | High, variable | Load |
| **Network** | Local/LAN | Cross-region/WAN | Latency |
| **Dependencies** | Mocked/stubbed | Real services | Reliability |
| **Configuration** | Dev settings | Production settings | Tuning |
| **Hardware** | Shared/VM | Dedicated/bare metal | Resources |
| **Security** | Relaxed | Strict | Access |

### 2. Check the Logs First

```bash
# Look for error patterns
grep -i "error\|exception\|timeout" application.log | tail -50

# Check for specific error types
grep "OutOfMemoryError" application.log
grep "Connection refused" application.log
grep "Timeout" application.log

# Check for rate limiting
grep "429\|Too Many Requests" application.log

# Check for circuit breaker trips
grep "CircuitBreaker" application.log
```

### 3. Common Root Causes

#### A. Data Volume Differences

```java
// Works with 100 records in staging, fails with 1M in production
public List<Result> processAll() {
    List<Data> allData = database.findAll();  // 1M records in prod!
    List<Result> results = new ArrayList<>();
    for (Data data : allData) {
        results.add(process(data));  // Takes hours in prod
    }
    return results;
}

// Fix: Pagination
public List<Result> processAll() {
    List<Result> results = new ArrayList<>();
    Pageable pageable = PageRequest.of(0, 1000);
    Page<Data> page;
    do {
        page = database.findAll(pageable);
        page.getContent().forEach(data -> 
            results.add(process(data)));
        pageable = page.nextPageable();
    } while (page.hasNext());
    return results;
}
```

#### B. Network Latency

```java
// Works in staging (local network), fails in production (cross-region)
public User getUser(String userId) {
    // 10ms in staging, 200ms in production
    return userClient.getUser(userId);
}

// Fix: Add timeouts and retries
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
public User getUser(String userId) {
    return userClient.getUser(userId);
}
```

#### C. Resource Limits

```yaml
# Staging: 2GB heap, 4 CPUs
# Production: 8GB heap, 16 CPUs — but different GC settings

# Fix: Ensure consistent JVM settings
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

#### D. Concurrency Issues

```java
// Race condition that only manifests under high load
public class Counter {
    private int count = 0;
    
    public void increment() {
        count++;  // Not thread-safe!
    }
}

// Fix: Use AtomicInteger or synchronized
public class SafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
}
```

#### E. External Service Differences

```java
// Staging uses mock services, production uses real ones
public PaymentResult processPayment(PaymentRequest request) {
    // In staging: returns immediately
    // In production: takes 5 seconds, sometimes times out
    return paymentGateway.process(request);
}

// Fix: Add timeouts and circuit breakers
public PaymentResult processPayment(PaymentRequest request) {
    return circuitBreaker.executeSupplier(
        TimeLimiter.decorateSupplier(
            timeoutExecutor,
            () -> paymentGateway.process(request)
        )
    );
}
```

#### F. Configuration Differences

```yaml
# Staging application.yml
database:
  pool-size: 5
  timeout: 30s

# Production application.yml
database:
  pool-size: 20
  timeout: 10s  # Too short for production load!
```

#### G. Security Restrictions

```java
// Works in staging (no firewall), fails in production (firewall)
public void connectToExternalService() {
    // Connection refused in production due to firewall rules
    socket.connect(new InetSocketAddress("external.com", 443));
}
```

### 4. Diagnostic Steps

#### Step 1: Reproduce in a Production-Like Environment

```bash
# Create a staging environment that mirrors production
# - Same data volume
# - Same network conditions
# - Same configuration
# - Same dependencies
```

#### Step 2: Load Testing

```bash
# Use JMeter, Gatling, or k6 to simulate production load
k6 run --vus 100 --duration 30s script.js

# Monitor metrics during load test
# - Response times
# - Error rates
# - Resource usage
```

#### Step 3: Chaos Engineering

```bash
# Introduce failures to test resilience
# - Kill random instances
# - Inject network latency
# - Simulate dependency failures
# Tools: Chaos Monkey, Gremlin
```

#### Step 4: Canary Deployment

```yaml
# Deploy to a small percentage of users first
# Monitor for issues before full rollout
canary:
  steps:
    - setWeight: 10  # 10% of traffic
    - pause: {duration: 1h}  # Monitor for 1 hour
    - setWeight: 50
    - pause: {duration: 1h}
    - setWeight: 100
```

### 5. Monitoring and Alerting

#### Key Metrics to Monitor

```java
// Application metrics
@Timed("api.response.time")
@Counted("api.requests")
@Observed(name = "order-processing")
public Order processOrder(OrderRequest request) {
    // Metrics automatically collected
}

// JVM metrics
// - Heap usage
// - GC frequency and pause times
// - Thread count
// - CPU usage

// Business metrics
// - Order success rate
// - Payment success rate
// - Error rates by type
```

#### Health Checks

```java
@Component
public class HealthCheck implements HealthIndicator {
    @Override
    public Health health() {
        // Check database connectivity
        // Check external service availability
        // Check disk space
        // Check memory usage
        return Health.up().build();
    }
}
```

## Key Takeaway

> When a service works in staging but fails in production, start by **comparing environments** — data volume, traffic, network, dependencies, and configuration. Check logs for error patterns, then systematically test hypotheses using load testing, chaos engineering, and canary deployments. The key is to **reproduce the production conditions** in a controlled environment and **monitor comprehensively** to catch issues before they affect users.
