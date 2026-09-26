# Monitoring Error Rates

## Key Metrics
- **Error rate percentage**: (errors / total requests) * 100
- **Error count by type**: 4xx, 5xx, exceptions
- **Error trends**: Over time
- **Endpoint-specific errors**

## Implementation
```java
@Component
public class ErrorMetrics {
    private final MeterRegistry registry;
    
    public ErrorMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public void recordError(String endpoint, String errorType) {
        Counter.builder("http.errors")
            .tag("endpoint", endpoint)
            .tag("type", errorType)
            .register(registry)
            .increment();
    }
}
```

## Alerting
```yaml
groups:
- name: api-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_errors_total[5m]) > 0.05
    for: 5m
    labels:
      severity: page
```

## Interview Tip
Explain that error rate monitoring should distinguish between client errors (4xx) and server errors (5xx) for better debugging.