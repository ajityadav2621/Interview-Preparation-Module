# Monitoring API Latency

## Key Metrics
- **Response time percentiles**: p50, p95, p99
- **Request throughput**: requests per second
- **Error rates**: 4xx, 5xx responses
- **Slow endpoints**: Identify problematic APIs

## Implementation
```java
@Component
public class LatencyMetrics {
    private final MeterRegistry registry;
    
    public LatencyMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public void recordLatency(String endpoint, long latencyMs) {
        Timer.builder("http.server.requests")
            .tag("uri", endpoint)
            .register(registry)
            .record(latencyMs, TimeUnit.MILLISECONDS);
    }
}
```

## Dashboard Queries
```promql
# P95 latency
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# Average latency
rate(http_server_requests_seconds_sum[5m]) / rate(http_server_requests_seconds_count[5m])
```

## Interview Tip
Explain that latency monitoring should focus on percentiles, not averages. P99 helps identify tail latency issues.