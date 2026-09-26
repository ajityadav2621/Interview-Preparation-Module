# Spring Boot Metrics to Monitor

## Key Metrics

### 1. **HTTP Metrics**
- Request count by status
- Response time percentiles (p50, p95, p99)
- Error rates
- Endpoint-specific metrics

### 2. **JVM Metrics**
- Heap usage
- Thread count
- Class loading
- GC performance

### 3. **System Metrics**
- CPU usage
- Memory usage
- Disk space
- File descriptor usage

### 4. **Database Metrics**
- Connection pool usage
- Query performance
- Active connections

## Accessing Metrics
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## Custom Metrics
```java
@Component
public class OrderMetrics {
    private final MeterRegistry registry;
    
    public OrderMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public void recordOrderCreated() {
        Counter.builder("orders.created")
            .register(registry)
            .increment();
    }
}
```

## Interview Tip
Explain that Spring Boot Actuator provides production-ready metrics out of the box. Customize with Micrometer for application-specific measurements.