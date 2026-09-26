# Trace Context Propagation

## What is Context Propagation?
Passing trace metadata (trace ID, span ID, etc.) between services in a distributed system.

## Propagation Methods

### 1. **HTTP Headers**
```java
// Incoming request
String traceId = request.getHeader("trace-id");
String spanId = request.getHeader("span-id");

// Outgoing request
headers.add("trace-id", currentTraceId);
headers.add("span-id", currentSpanId);
```

### 2. **W3C Trace Context**
```java
// Standard headers
headers.add("traceparent", "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01");
```

### 3. **B3 Propagation**
```java
// Zipkin B3 headers
headers.add("X-B3-TraceId", traceId);
headers.add("X-B3-SpanId", spanId);
headers.add("X-B3-ParentSpanId", parentSpanId);
```

## Interview Tip
Explain that context propagation is critical for distributed tracing. W3C Trace Context is becoming the industry standard.