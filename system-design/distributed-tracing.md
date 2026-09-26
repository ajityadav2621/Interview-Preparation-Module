# A Request Travels Through 6 Microservices. How Would You Trace It End-to-End?

## The Problem

In a microservices architecture, a single user request may travel through multiple services. When something goes wrong, it's difficult to understand the full request flow and identify the bottleneck.

## The Solution: Distributed Tracing

**Distributed tracing** tracks a request as it flows through multiple services, providing end-to-end visibility.

## How Distributed Tracing Works

### Core Concepts

1. **Trace**: A distributed transaction — a single user request flowing through multiple services
2. **Span**: A single operation within a trace — a method call, HTTP request, database query
3. **Trace Context**: Metadata that connects spans across services (trace ID, span ID, parent span ID)

```
Trace: GET /checkout (Trace ID: abc123)
├── Span 1: Order Service - createOrder (Span ID: 1, Parent: none)
│   ├── Span 2: User Service - getUser (Span ID: 2, Parent: 1)
│   ├── Span 3: Payment Service - processPayment (Span ID: 3, Parent: 1)
│   │   └── Span 4: External API - chargeCard (Span ID: 4, Parent: 3)
│   └── Span 5: Inventory Service - reserveItems (Span ID: 5, Parent: 1)
└── Span 6: Order Service - sendConfirmation (Span ID: 6, Parent: 1)
```

## Implementation with OpenTelemetry

### 1. Setup OpenTelemetry SDK

```xml
<!-- Maven dependencies -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.32.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
    <version>1.32.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <version>1.32.0</version>
</dependency>
```

### 2. Configure Tracer

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;

@Configuration
public class TracingConfig {
    
    @Bean
    public Tracer tracer() {
        return GlobalOpenTelemetry.getTracer("my-app", "1.0.0");
    }
}
```

### 3. Instrument Services

```java
@RestController
public class OrderController {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PaymentService paymentService;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        Span span = tracer.spanBuilder("createOrder")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("order.id", request.getOrderId());
            span.setAttribute("user.id", request.getUserId());
            
            // Call other services — trace context is automatically propagated
            User user = userService.getUser(request.getUserId());
            PaymentResult payment = paymentService.processPayment(request.getPayment());
            
            Order order = new Order(request, user, payment);
            return ResponseEntity.ok(order);
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 4. Propagate Trace Context

```java
@Service
public class UserService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public User getUser(String userId) {
        Span span = tracer.spanBuilder("getUser")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("user.id", userId);
            
            // Trace context is automatically added to HTTP headers
            ResponseEntity<User> response = restTemplate.getForEntity(
                "http://user-service/api/users/" + userId, User.class);
            
            return response.getBody();
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 5. HTTP Header Propagation

```java
// OpenTelemetry automatically propagates trace context via HTTP headers
// W3C Trace Context headers:
// traceparent: 00-abc123def456-01-01
// tracestate: congo=t61rcWkgMzE

// When making HTTP calls, the trace context is automatically added:
// GET /api/users/123
// Headers:
//   traceparent: 00-abc123def456-01-02
//   tracestate: congo=t61rcWkgMzE
```

## Using Spring Cloud Sleuth (Simpler)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```java
@RestController
public class OrderController {
    
    @Autowired
    private UserService userService;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // Sleuth automatically creates spans for each service call
        User user = userService.getUser(request.getUserId());
        PaymentResult payment = paymentService.processPayment(request.getPayment());
        
        return ResponseEntity.ok(new Order(request, user, payment));
    }
}
```

## Visualization with Jaeger

### Setup Jaeger

```yaml
# docker-compose.yml
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # OTLP gRPC
      - "4317:4317"    # OTLP HTTP
```

### View Traces

1. Open `http://localhost:16686`
2. Search for traces by service name, operation, or trace ID
3. View the trace timeline to see which spans are slow

## Key Metrics to Track

### 1. Trace Latency

```
Trace: createOrder
├── getUser: 50ms
├── processPayment: 200ms
│   └── chargeCard: 180ms  ← BOTTLENECK!
└── reserveItems: 30ms
Total: 280ms
```

### 2. Error Rates

```
Service: Order Service
- Total requests: 1000
- Errors: 5
- Error rate: 0.5%

Service: Payment Service
- Total requests: 1000
- Errors: 50
- Error rate: 5%  ← Investigate!
```

### 3. Service Dependencies

```
Order Service
├── User Service (95th percentile: 50ms)
├── Payment Service (95th percentile: 200ms)
└── Inventory Service (95th percentile: 30ms)
```

## Best Practices

### 1. Add Meaningful Attributes

```java
span.setAttribute("http.method", "POST");
span.setAttribute("http.url", request.getRequestURL().toString());
span.setAttribute("user.id", userId);
span.setAttribute("order.id", orderId);
```

### 2. Record Exceptions

```java
try {
    processPayment();
} catch (PaymentException e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, "Payment failed");
    throw e;
}
```

### 3. Use Semantic Conventions

```java
// Follow OpenTelemetry semantic conventions
span.setAttribute("db.system", "postgresql");
span.setAttribute("db.statement", "SELECT * FROM orders WHERE id = ?");
span.setAttribute("db.name", "ecommerce");
```

### 4. Sample Appropriately

```yaml
# Don't trace every request — sample to reduce overhead
otel:
  traces:
    sampler: parentbased_traceidratio
    ratio: 0.1  # Sample 10% of requests
```

## Key Takeaway

> Distributed tracing provides end-to-end visibility into requests flowing through multiple microservices. Use **OpenTelemetry** (or Spring Cloud Sleuth) to instrument your services, propagate trace context via HTTP headers, and visualize traces with **Jaeger** or **Zipkin**. The key is to track **trace IDs** across service boundaries, record **meaningful attributes** and **exceptions**, and use the traces to identify **bottlenecks** and **errors** in the request flow.
