# Handling Sudden Traffic Spike

## Strategies

### 1. **Auto-scaling**
```yaml
# Kubernetes HPA configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: application-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: application
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 2. **Rate Limiting**
```java
// Rate limiting per client/IP
Bucket bucket = Bucket.builder()
    .addLimit(Bandwidth.classic(1000, Duration.ofMinutes(1))
    .build();

if (bucket.tryConsume(1)) {
    // Process request
} else {
    // Return 429 Too Many Requests
}
```

### 3. **Caching**
```java
// Cache frequently accessed data
@Cacheable(value = "products", key = "#productId")
public Product getProduct(Long productId) {
    return productRepository.findById(productId)
        .orElseThrow();
}
```

### 4. **Queue-Based Load Leveling**
```java
// Use message queue for async processing
@Service
public class OrderService {
    
    @Async
    public CompletableFuture<Void> processOrderAsync(Order order) {
        orderRepository.save(order);
        return CompletableFuture.completedFuture(null);
    }
}
```

## Interview Tip
Explain that traffic spikes require multiple layers of protection: caching, rate limiting, auto-scaling, and async processing.