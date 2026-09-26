# Tracing Requests Across Multiple Services

## Distributed Tracing Concepts

### 1. **Trace ID and Span ID**
```
Trace ID: abc123def456 (unique for entire request)
  Span ID: 111 (service A)
    Span ID: 222 (service B)
      Span ID: 333 (service C)
```

### 2. **Context Propagation**
```java
// Propagate trace context
public void callService() {
    SpanContext context = Tracer.currentSpan().context();
    
    // Add to HTTP headers
    headers.add("trace-id", context.traceId());
    headers.add("span-id", context.spanId());
    
    // Make HTTP call
    RestTemplate.getForObject(url, String.class, headers);
}
```

## Implementation

### 1. **OpenTelemetry Integration**
```java
dependencies {
    implementation 'io.opentelemetry:opentelemetry-api:1.30.0'
    implementation 'io.opentelemetry:opentelemetry-sdk:1.30.0'
    implementation 'io.opentelemetry:opentelemetry-auto-annotation:1.30.0'
}

// Auto-instrumentation
opentelemetry:
  java:
    enabled: true
```

### 2. **Manual Tracing**
```java
@NewSpan
public void processOrder(String orderId) {
    Span span = Tracer.currentSpan();
    span.setAttribute("order.id", orderId);
    
    // ... process order
    
    span.addEvent("Order processed");
    span.setStatus(StatusCode.OK);
}

@SpanAttribute
public Order processOrder(@SpanAttribute String orderId) {
    return orderService.create(orderId);
}
```

## Interview Tip
Explain that distributed tracing helps visualize request flows. OpenTelemetry is becoming the standard for tracing implementation.