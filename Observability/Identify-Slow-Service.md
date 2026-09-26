# Identifying Slow Microservice with Distributed Tracing

## Approach

### 1. **Analyze Trace Data**
```java
// Find slow spans in a trace
Trace trace = tracer.getTrace(traceId);
for (Span span : trace.getSpans()) {
    long duration = span.getDuration().toMillis();
    if (duration > threshold) {
        log.warn("Slow service: {} took {}ms", 
                span.getServiceName(), duration);
    }
}
```

### 2. **Visualize Trace Flow**
```
Trace: abc123 (2.5s total)
  API Gateway: 50ms
  Order Service: 2000ms ← BOTTLENECK
    Database: 150ms
    Payment Service: 1800ms ← SUB-BOTTLENECK
  Shipping Service: 450ms
```

### 3. **Analyze Span Timings**
- Identify service with longest duration
- Check for nested slow operations
- Look for parallel vs sequential execution

## Interview Tip
Explain that distributed tracing helps visualize where time is spent across services. The longest span usually indicates the bottleneck.