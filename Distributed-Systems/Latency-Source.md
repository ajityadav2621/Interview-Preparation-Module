# Identifying Latency-Causing Service

## Approach

### 1. **Distributed Tracing Analysis**
```java
// Analyze trace data
Trace trace = tracer.getTrace(traceId);
for (Span span : trace.getSpans()) {
    long duration = span.getDuration().toMillis();
    if (duration > threshold) {
        log.warn("Slow service: {} took {}ms", 
                 span.getServiceName(), duration);
    }
}
```

### 2. **Metrics Analysis**
- Service response times
- Dependency latency
- Error rates
- Resource utilization

### 3. **Log Correlation**
```bash
# Correlate logs with trace IDs
grep "trace_id=abc123" application.log

# Analyze timing
grep "trace_id=abc123" application.log | awk '{print $1, $NF}'
```

### 4. **Performance Profiling**
```bash
# Profile slow services
jstack <pid> | grep -A 30 "BLOCKED"
jstat -gc <pid>
```

## Interview Tip
Explain that identifying the bottleneck requires correlating tracing data with metrics and logs. Start with distributed tracing to visualize the flow.