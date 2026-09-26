# Database Queries Are Fast, but the API Is Slow. What Else Could Be Causing the Latency?

## The Problem

When database queries are fast but the API is slow, the bottleneck is **not** the database. The latency is coming from somewhere else in the request processing pipeline.

## Common Causes

### 1. Network Latency

#### External API Calls
```java
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        Order order = orderService.getOrder(id);           // Fast DB query
        User user = userService.getUser(order.getUserId()); // Slow external API call!
        return new OrderResponse(order, user);
    }
}
```

**Diagnosis**:
- Check network latency to external services
- Use distributed tracing to identify slow spans
- Monitor external API response times

#### DNS Resolution
```bash
# Check DNS resolution time
dig api.external-service.com
nslookup api.external-service.com

# DNS can add 50-500ms per lookup
```

### 2. Thread Pool Exhaustion

```java
// If the thread pool is exhausted, requests queue up
@RestController
public class SlowController {
    
    @Autowired
    private ExecutorService executor;  // Fixed thread pool of 10
    
    @GetMapping("/slow")
    public CompletableFuture<String> slowEndpoint() {
        return CompletableFuture.supplyAsync(() -> {
            // If all 10 threads are busy, this queues
            return externalService.call();  // Takes 2 seconds
        }, executor);
    }
}
```

**Diagnosis**:
- Check thread pool metrics (active, queued, completed)
- Monitor request queue depth
- Look for `RejectedExecutionException`

### 3. Connection Pool Exhaustion

```java
// If all database connections are in use, new requests wait
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(10);  // Only 10 connections!
        return new HikariDataSource(config);
    }
}
```

**Diagnosis**:
- Monitor connection pool metrics
- Check for connection leaks
- Look for `Connection is not available` errors

### 4. Serialization/Deserialization Overhead

```java
// Large JSON payloads can be slow to serialize/deserialize
@RestController
public class DataController {
    
    @GetMapping("/large-data")
    public List<LargeObject> getLargeData() {
        List<LargeObject> data = database.fetchMillionRecords();
        // Serializing 1M objects to JSON can take seconds!
        return data;
    }
}
```

**Diagnosis**:
- Profile serialization time
- Check payload sizes
- Consider pagination or streaming

### 5. Caching Issues

```java
// Cache miss causes slow response
@RestController
public class CachedController {
    
    @Cacheable("users")
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        // First request: slow (cache miss)
        // Subsequent requests: fast (cache hit)
        return database.findUser(id);
    }
}
```

**Diagnosis**:
- Monitor cache hit ratio
- Check cache eviction patterns
- Look for cache stampede

### 6. Garbage Collection Pauses

```java
// GC pauses can cause latency spikes
@RestController
public class GcSensitiveController {
    
    @GetMapping("/process")
    public Result process() {
        // If a Full GC happens during this request,
        // the response time will spike
        return heavyProcessing();
    }
}
```

**Diagnosis**:
- Check GC logs for pause times
- Monitor heap usage
- Look for `Full GC` events

### 7. Lock Contention

```java
// Synchronized blocks can cause thread contention
@Service
public class ContendedService {
    
    private final Object lock = new Object();
    
    public void process() {
        synchronized (lock) {
            // If many threads compete for this lock,
            // they'll queue up and cause latency
            doWork();
        }
    }
}
```

**Diagnosis**:
- Thread dumps showing BLOCKED threads
- Monitor lock contention metrics
- Use `jstack` to identify contention points

### 8. I/O Bottlenecks

#### Disk I/O
```java
// Reading large files from disk
@GetMapping("/report")
public Report generateReport() {
    // Reading a 1GB file from disk can be slow
    List<Data> data = fileReader.readLargeFile("/data/report.csv");
    return process(data);
}
```

#### Network I/O
```java
// Multiple sequential external calls
@GetMapping("/dashboard")
public Dashboard getDashboard() {
    User user = userService.getUser(userId);           // 200ms
    List<Order> orders = orderService.getOrders(userId); // 300ms
    List<Notification> notifications = notificationService.get(userId); // 150ms
    // Total: 650ms — could be parallelized!
    return new Dashboard(user, orders, notifications);
}
```

### 9. Third-Party Service Latency

```java
// External service is slow
@Service
public class PaymentService {
    public PaymentResult process(PaymentRequest request) {
        // External payment gateway takes 5 seconds
        return paymentGateway.charge(request);
    }
}
```

**Diagnosis**:
- Monitor external service response times
- Check for rate limiting
- Look for service degradation

### 10. Application-Level Bottlenecks

#### Synchronous Processing
```java
// Processing should be async
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestBody FileData data) {
    // This blocks the request thread for a long time
    processFile(data);  // Takes 10 seconds
    return ResponseEntity.ok("Done");
}
```

#### Inefficient Algorithms
```java
// O(n²) algorithm on large dataset
public List<Result> process(List<Input> inputs) {
    List<Result> results = new ArrayList<>();
    for (Input input : inputs) {
        for (Input other : inputs) {  // O(n²) — slow for large lists!
            if (input.matches(other)) {
                results.add(process(input, other));
            }
        }
    }
    return results;
}
```

## Diagnostic Approach

### 1. Distributed Tracing
```java
// Use OpenTelemetry or similar
@Traced
@GetMapping("/orders/{id}")
public OrderResponse getOrder(@PathVariable String id) {
    // Each service call is traced
    // Can see exactly where time is spent
}
```

### 2. Application Metrics
```java
// Use Micrometer or similar
@RestController
public class MonitoredController {
    
    @Timed("api.response.time")
    @Counted("api.requests")
    @GetMapping("/orders/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        // Metrics automatically collected
    }
}
```

### 3. Profiling
```bash
# CPU profiling
jcmd <pid> VM.native_memory summary

# Or use async-profiler
./profiler.sh -e cpu -d 30 -f profile.html <pid>
```

## Key Takeaway

> When database queries are fast but the API is slow, the bottleneck is likely in **network calls** (external APIs, DNS), **resource exhaustion** (thread pools, connection pools), **serialization overhead**, **GC pauses**, **lock contention**, or **inefficient algorithms**. Use **distributed tracing** to identify the exact source of latency, and **application metrics** to monitor resource usage. The key is to measure, not guess.
