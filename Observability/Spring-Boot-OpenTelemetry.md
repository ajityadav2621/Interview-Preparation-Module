# Spring Boot and OpenTelemetry Integration

## Auto-instrumentation
```yaml
# application.yml
opentelemetry:
  java:
    enabled: true
  traces:
    sampler:
      ratio: 1.0
```

## Dependencies
```java
dependencies {
    implementation 'io.opentelemetry:opentelemetry-auto-annotation:1.30.0'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.30.0'
}
```

## Manual Tracing
```java
@NewSpan
public void processOrder(@SpanAttribute String orderId) {
    Span span = Tracer.currentSpan();
    span.setAttribute("order.id", orderId);
    span.addEvent("Order processing started");
}
```

## Interview Tip
Explain that OpenTelemetry provides both auto-instrumentation for common frameworks and manual tracing for custom code.