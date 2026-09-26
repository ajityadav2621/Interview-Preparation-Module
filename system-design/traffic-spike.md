# One Service Is Receiving Much More Traffic Than Others. How Would You Handle It?

## The Problem

In a microservices architecture, traffic is often unevenly distributed. One service may receive significantly more traffic than others, leading to:

- High latency
- Resource exhaustion
- Cascading failures
- Poor user experience

## Solutions

### 1. Horizontal Scaling (Scale Out)

Add more instances of the overloaded service:

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 10  # Scale from 3 to 10
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

### 2. Load Balancing

```yaml
# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 3. Rate Limiting

Protect the service from being overwhelmed:

```java
@RestController
public class OrderController {
    
    @RateLimiter(name = "order-service", fallbackMethod = "rateLimitFallback")
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        return orderService.create(request);
    }
    
    public ResponseEntity<Order> rateLimitFallback(OrderRequest request, Exception ex) {
        return ResponseEntity.status(429).build();  // Too Many Requests
    }
}
```

```yaml
# Resilience4j rate limiter configuration
resilience4j:
  ratelimiter:
    instances:
      order-service:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout: 0
```

### 4. Caching

Reduce the load on the service by caching responses:

```java
@Service
public class OrderService {
    
    @Cacheable(value = "orders", key = "#orderId")
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId);
    }
    
    @CacheEvict(value = "orders", key = "#order.id")
    public Order updateOrder(Order order) {
        return orderRepository.save(order);
    }
}
```

```yaml
# Redis cache configuration
spring:
  redis:
    host: redis-cluster
    port: 6379
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
```

### 5. Circuit Breaker

Prevent cascading failures when the service is overwhelmed:

```java
@Service
public class OrderService {
    
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("order-service");
    
    public Order getOrder(String orderId) {
        return circuitBreaker.executeSupplier(() -> 
            orderRepository.findById(orderId));
    }
}
```

### 6. Asynchronous Processing

Move heavy processing to background:

```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // Quick response — process in background
        CompletableFuture.supplyAsync(() -> {
            return orderService.processOrder(request);
        });
        
        return ResponseEntity.accepted().build();
    }
}
```

### 7. Database Optimization

```sql
-- Add indexes for frequently queried columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Use read replicas for read-heavy operations
SELECT * FROM orders_read_replica WHERE user_id = ?;
```

### 8. Content Delivery Network (CDN)

For static content, use a CDN:

```yaml
# CDN configuration
cdn:
  enabled: true
  origin: https://api.example.com
  ttl: 3600
  cache-key:
    include-host: true
    include-protocol: true
```

### 9. Request Queuing

Queue requests when the service is overloaded:

```java
@Service
public class OrderService {
    
    private final ThreadPoolBulkhead bulkhead = 
        ThreadPoolBulkhead.of("order-service", 
            ThreadPoolBulkheadConfig.custom()
                .coreThreadPoolSize(20)
                .maxThreadPoolSize(50)
                .queueCapacity(100)
                .build());
    
    public CompletableFuture<Order> processOrder(OrderRequest request) {
        return bulkhead.executeSupplier(() -> {
            return orderProcessor.process(request);
        });
    }
}
```

### 10. Auto-scaling

Automatically scale based on metrics:

```yaml
# Kubernetes Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Traffic Management Strategies

### 1. Priority-Based Processing

```java
public enum Priority {
    HIGH, NORMAL, LOW
}

public class PriorityOrderProcessor {
    private final PriorityBlockingQueue<Order> highPriorityQueue = 
        new PriorityBlockingQueue<>(11, Comparator.comparing(Order::getPriority));
    private final PriorityBlockingQueue<Order> normalPriorityQueue = 
        new PriorityBlockingQueue<>(11, Comparator.comparing(Order::getPriority));
    
    public void processOrder(Order order) {
        if (order.getPriority() == Priority.HIGH) {
            highPriorityQueue.offer(order);
        } else {
            normalPriorityQueue.offer(order);
        }
    }
}
```

### 2. Load Shedding

```java
@Service
public class LoadSheddingService {
    
    private final AtomicLong requestCount = new AtomicLong(0);
    private static final int MAX_REQUESTS_PER_SECOND = 1000;
    
    public boolean shouldProcess() {
        long count = requestCount.incrementAndGet();
        if (count > MAX_REQUESTS_PER_SECOND) {
            return false;  // Shed load
        }
        return true;
    }
}
```

### 3. Request Prioritization

```java
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(
            @RequestBody OrderRequest request,
            @RequestHeader(value = "X-Priority", defaultValue = "NORMAL") String priority) {
        
        if ("LOW".equals(priority) && isOverloaded()) {
            return ResponseEntity.status(503).build();  // Service Unavailable
        }
        
        return orderService.create(request);
    }
}
```

## Monitoring and Alerting

```yaml
# Prometheus metrics
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

# Alert rules
groups:
- name: service-alerts
  rules:
  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High latency on {{ $labels.service }}"
```

## Key Takeaway

> When one service receives much more traffic, use a **combination of strategies**: **horizontal scaling** (add instances), **rate limiting** (protect from overload), **caching** (reduce load), **circuit breakers** (prevent cascading failures), **asynchronous processing** (move heavy work to background), and **auto-scaling** (automatically adjust capacity). Monitor traffic patterns and set up alerts to detect issues before they impact users.
