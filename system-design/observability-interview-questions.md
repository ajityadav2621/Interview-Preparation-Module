# Observability Interview Questions

## 1. What is Observability?

**Observability** is the ability to understand the internal state of a system by examining its external outputs. In software systems, observability enables you to:

- **Detect** issues before they impact users
- **Diagnose** the root cause of problems quickly
- **Understand** system behavior under different conditions
- **Validate** that the system is functioning correctly

Observability goes beyond traditional monitoring by providing rich, contextual data that allows you to explore unknown failure modes rather than just alerting on predefined conditions.

---

## 2. What is the Difference Between Monitoring and Observability?

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| **Approach** | Reactive, predefined dashboards | Proactive, exploratory analysis |
| **Data** | Fixed metrics and thresholds | Dynamic, high-cardinality data |
| **Questions** | "Is metric X above threshold?" | "Why is metric X behaving this way?" |
| **Failure Detection** | Known failure patterns | Both known and unknown patterns |
| **Debugging** | Limited context for root cause | Rich context enabling deep dive |
| **Scope** | System health monitoring | End-to-end system understanding |

**Key Difference**: Monitoring tells you **if** something is wrong, while observability helps you understand **why** something is wrong and **how** to fix it.

---

## 3. What are the Three Pillars of Observability?

The three pillars of observability are:

### 1. **Metrics**
- Quantitative measurements over time
- Examples: CPU usage, request count, response times
- Stored as time-series data
- Good for alerting and dashboards

### 2. **Logs**
- Discrete events with timestamps
- Detailed event information
- Examples: ERROR messages, transaction details
- Good for debugging and auditing

### 3. **Traces**
- End-to-end request paths across services
- Shows causality between operations
- Examples: HTTP requests across microservices
- Good for distributed system debugging

---

## 4. Metrics vs Logs vs Traces

### Metrics
**Characteristics:**
- Low cardinality
- Aggregated numerical data
- Efficient storage
- Ideal for dashboards and alerts
- Examples: Counters, Gauges, Histograms

### Logs
**Characteristics:**
- High granularity
- Human-readable messages
- Higher storage costs
- Ideal for debugging
- Examples: Application logs, Audit logs, Access logs

### Traces
**Characteristics:**
- Shows request flow
- Causal relationships
- Critical for distributed systems
- Ideal for latency analysis
- Examples: Distributed traces, Span data

---

## 5. What is Micrometer?

**Micrometer** is a metrics instrumentation library that provides a vendor-neutral facade for collecting application metrics in Java applications.

**Key Features:**
- **Facade Pattern**: Write metrics once, send to any backend
- **Multi-dimensional Metrics**: Tags for filtering and grouping
- **Common Integrations**: Spring Boot, Reactor, WebFlux
- **Auto-configuration**: Works seamlessly with Spring Boot Actuator

**Supported Backends:**
- Prometheus
- Datadog
- New Relic
- CloudWatch
- Graphite
- InfluxDB

---

## 6. Why is Micrometer used in Spring Boot?

Micrometer is the de facto standard for metrics in Spring Boot because:

### 1. **Spring Boot Actuator Integration**
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

### 2. **Auto-configuration**
Spring Boot auto-configures Micrometer with sensible defaults when `micrometer-registry-prometheus` is on the classpath.

### 3. **Meter Registry Abstraction**
```java
@RestController
public class OrderController {
    
    private final MeterRegistry meterRegistry;
    
    public OrderController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            return orderService.findById(id);
        } finally {
            sample.stop(Timer.builder("order.fetch")
                .tag("type", "http")
                .register(meterRegistry));
        }
    }
}
```

### 4. **Annotation-based Metrics**
```java
@Timed(value = "order.create", description = "Time to create order")
@Counted(value = "order.created", description = "Number of orders created")
public Order createOrder(OrderRequest request) {
    return orderService.save(request);
}
```

---

## 7. What is OpenTelemetry?

**OpenTelemetry** (OTel) is an open-source observability framework that provides standardized collection and export of:

- **Traces** (distributed tracing)
- **Metrics** (numerical measurements)
- **Logs** (event records)

**Key Components:**
1. **Collector**: Processes and exports telemetry data
2. **SDK**: Language-specific implementations
3. **Semantic Conventions**: Standard attribute names
4. **Auto-instrumentation**: Zero-code instrumentation for many frameworks

**Benefits:**
- Vendor neutrality
- Single instrumentation point
- Active community and CNCF project
- Supports legacy and modern applications

---

## 8. OpenTelemetry vs Micrometer

| Aspect | OpenTelemetry | Micrometer |
|--------|---------------|------------|
| **Primary Focus** | Traces, Metrics, Logs | Metrics |
| **Scope** | End-to-end observability | Metrics facade |
| **Tracing Support** | Native support | Via Micrometer Tracing |
| **Vendor Support** | Built-in exporters | Registry-based backends |
| **Complexity** | More complex setup | Simpler for metrics-only |
| **Use Case** | Full observability stack | Metrics aggregation |

### When to Use Each:

**Micrometer Alone:**
- Simple applications
- Metrics-only requirements
- Quick implementation
- Spring Boot applications

**OpenTelemetry:**
- Distributed systems requiring traces
- Multi-signal observability
- Long-term vendor neutrality
- Complex microservices architecture

---

## 9. What is Distributed Tracing?

**Distributed tracing** is the process of tracking requests as they flow through multiple services in a distributed system.

**Key Concepts:**
1. **Trace**: The entire journey of a request across services
2. **Span**: A single unit of work within that trace
3. **Context Propagation**: Passing trace information between services

**Benefits:**
- Identify latency bottlenecks
- Understand request flow
- Debug cross-service failures
- Performance optimization

---

## 10. What is a Trace ID?

A **Trace ID** is a unique identifier that groups all spans related to a single request across multiple services.

**Characteristics:**
- 32-character hex string (128 bits)
- Generated at request entry point
- Propagated via HTTP headers
- Same for all spans in a request

**W3C Trace Context format:**
```
traceparent: 00-{traceId}-{spanId}-{flags}
Example: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

---

## 11. What is a Span?

A **Span** represents a single operation or unit of work within a trace.

**Span Attributes:**
- **Name**: The operation name
- **Start/End Time**: When it occurred
- **Parent Span ID**: The calling span
- **Status**: Success or error
- **Events**: Timestamped annotations
- **Links**: Connections to other spans

**Span Lifecycle:**
1. Span Created (startSpan)
2. Work Executed (doWork)
3. Events Added (addEvent)
4. Span Ended (span.end)

---

## 12. Trace ID vs Span ID

| Aspect | Trace ID | Span ID |
|--------|----------|---------|
| **Purpose** | Groups related spans | Identifies individual operations |
| **Scope** | Entire request | Single service/operation |
| **Generation** | At request entry | At span creation |
| **Propagation** | Propagated to all services | Each service generates its own |
| **Uniqueness** | Unique per request | Unique per span |
| **Visibility** | Request-scoped | Operation-scoped |

---

## 13. How does Trace Context Propagate Between Microservices?

Trace context propagation ensures that trace information flows correctly across service boundaries.

### HTTP Header Propagation (W3C Trace Context)
```
# Outgoing HTTP Request Headers
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: congo=t61rcWkgMzE
```

### Propagation Mechanisms:
1. **W3C Trace Context**: Standard HTTP headers (recommended)
2. **B3 Propagation**: Zipkin format
3. **Jaeger**: Jaeger-specific headers
4. **gRPC**: Metadata propagation

### Spring Boot with OpenTelemetry
```java
// Automatic propagation with RestTemplate or WebClient
@RestController
public class OrderController {
    
    @Autowired
    private RestTemplate restTemplate; // Auto-instrumented
    
    public Order getOrderFromInventory(Long orderId) {
        // Trace context is automatically propagated
        return restTemplate.getForObject(
            "http://inventory-service/items/" + orderId, 
            Order.class
        );
    }
}
```

---

## 14. How does Spring Boot Integrate with OpenTelemetry?

### Dependencies
```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

### Configuration
```yaml
# application.yml
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  service:
    name: ${spring.application.name}
```

### Manual Instrumentation
```java
@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return AutoConfiguredOpenTelemetrySdk.initialize()
            .getOpenTelemetrySdk();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("my-service");
    }
}
```

### Auto-instrumentation covers:
- HTTP requests (servlet, REST controllers)
- JDBC calls
- Redis operations
- Kafka messages
- Spring beans

---

## 15. How does Prometheus Collect Metrics?

Prometheus uses a **pull-based model** to collect metrics from targets.

### Collection Process:
1. **Scrape Loop**: Prometheus runs a scrape loop at regular intervals
2. **HTTP Endpoint**: Targets expose metrics at `/metrics`
3. **Data Retrieval**: Prometheus pulls data via HTTP GET
4. **Storage**: Metrics stored in time-series database

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
    scrape_interval: 15s
```

### Scrape Response Format
```
# HELP jvm_memory_used_bytes JVM memory usage
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="eden"} 1.234e8
```

---

## 16. What is the Prometheus Pull Model?

The **Prometheus Pull Model** is a metric collection approach where Prometheus actively retrieves metrics from targets.

### Advantages:
- **No agent on applications**: Simpler deployment
- **Target discovery**: Easy to add/remove targets
- **Health checks**: Can verify target availability
- **Resource efficiency**: Prometheus controls scrape rate

### Comparison with Push Model:
| Aspect | Pull Model | Push Model |
|--------|-----------|------------|
| **Initiation** | Server pulls | Client pushes |
| **Complexity** | Server-side | Client-side |
| **Reliability** | Server controls | Client dependent |
| **Use Case** | Long-running services | Short-lived jobs |

---

## 17. What is Grafana Used For?

**Grafana** is an open-source analytics and visualization platform for observability data.

### Key Features:
1. **Dashboards**: Create rich, interactive visualizations
2. **Data Sources**: Connect to Prometheus, Elasticsearch, InfluxDB, etc.
3. **Alerting**: Configure alerts based on metrics
4. **Templates**: Reusable dashboard components

### Common Visualizations:
- Time series graphs
- Stat panels
- Heatmaps
- Tables
- Alert status panels

---

## 18. Prometheus vs Grafana

| Aspect | Prometheus | Grafana |
|--------|------------|---------|
| **Primary Function** | Metrics collection & storage | Visualization |
| **Data Storage** | Time-series database | Query multiple sources |
| **Query Language** | PromQL | Multiple (PromQL, SQL, etc.) |
| **Type** | Metrics backend | Analytics platform |
| **Alerting** | Built-in alerting | Built-in alerting |
| **Relationship** | Often used together | Works with many backends |

### Recommended Architecture:
```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│   Spring     │────→│  Prometheus │────→│   Grafana    │
│   Boot App   │     │   Server    │     │   Dashboard  │
└──────────────┘     └─────────────┘     └──────────────┘
```

---

## 19. What JVM Metrics Should You Monitor?

### Memory Metrics
```
jvm_memory_used_bytes{area="heap"}      # Heap memory usage
jvm_memory_max_bytes{area="heap"}       # Max heap memory
jvm_memory_used_bytes{area="nonheap"}  # Non-heap memory
```

### GC Metrics
```
jvm_gc_pause_seconds_count  # GC count
jvm_gc_pause_seconds_sum    # Total GC time
jvm_gc_pause_seconds_bucket  # For histograms
```

### Thread Metrics
```
jvm_threads_live_threads      # Current thread count
jvm_threads_peak_threads      # Peak thread count
jvm_threads_states_threads{state="runnable"}   # Runnable threads
jvm_threads_states_threads{state="waiting"}    # Waiting threads
```

### Class Loading Metrics
```
jvm_classes_loaded           # Currently loaded classes
jvm_classes_unloaded_total   # Classes unloaded
```

---

## 20. What Spring Boot Metrics Should You Monitor?

### HTTP Metrics
```
http_server_requests_seconds_count    # Request count
http_server_requests_seconds_sum      # Total request time
http_server_requests_seconds_bucket   # Latency distribution
```

### Database Metrics (HikariCP)
```
hikaricp_connections_active           # Active connections
hikaricp_connections_idle            # Idle connections
hikaricp_connections_pending         # Pending requests
hikaricp_connections_timeout_total    # Timeout count
```

### Cache Metrics
```
cache_gets_total{hit="true"}          # Cache hits
cache_gets_total{hit="false"}         # Cache misses
cache_size                           # Cache size
```

### Kafka Metrics
```
kafka_consumer_records_lag_max       # Consumer lag
kafka_consumer_fetch_manager_records_consumed_total
```

---

## 21. How do you Monitor API Latency?

### Using Micrometer Timer
```java
@RestController
public class OrderController {
    
    private final MeterRegistry meterRegistry;
    
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            return orderService.findById(id);
        } finally {
            sample.stop(Timer.builder("api.orders.latency")
                .tag("endpoint", "getOrder")
                .register(meterRegistry));
        }
    }
}
```

### Key Metrics to Track:
```promql
# Average latency
rate(http_server_requests_seconds_sum[5m]) / 
rate(http_server_requests_seconds_count[5m])

# p95 latency
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))
```

---

## 22. How do you Monitor Error Rates?

### Counter-based Error Tracking
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private final MeterRegistry meterRegistry;
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<?> handleOrderNotFound(OrderNotFoundException ex) {
        meterRegistry.counter("orders.errors", 
            "type", "not_found"
        ).increment();
        return ResponseEntity.status(404).body(ex.getMessage());
    }
}
```

### Key Metrics:
```promql
# Error rate percentage
100 * sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / 
sum(rate(http_server_requests_seconds_count[5m]))
```

---

## 23. What is p95, p99 Latency?

### Percentiles Explained:
- **p50 (Median)**: 50% of requests are faster than this value
- **p95**: 95% of requests are faster than this value
- **p99**: 99% of requests are faster than this value

### Example:
```
# 100 requests with latencies:
# p50 = 50ms    (median)
# p95 = 500ms   (95th percentile)
# p99 = 2000ms  (99th percentile)
```

### Why Percentiles Matter:
p50 looks good, but p99 reveals tail latency issues that affect user experience!

---

## 24. Why is Average Latency Sometimes Misleading?

### The Problem with Averages:
```java
// Example: 100 requests
// 99 requests take 10ms
// 1 request takes 10000ms

// Average: 109.9ms  ← Looks acceptable!
// p99: 10000ms      ← Users experiencing slowness!
```

### Why Averages Fail:
1. **Skewed by outliers**: Extreme values distort the average
2. **Hides distribution**: Doesn't show variability
3. **Doesn't reflect user experience**: Most users get 10ms, some get 10 seconds

### Better Metrics:
```promql
# Use percentiles instead of averages
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))
```

---

## 25. How do you Identify a Slow Microservice Using Distributed Tracing?

### Step 1: Identify the Slow Trace
Find traces with high total duration in your tracing UI (Jaeger, Zipkin, etc.)

### Step 2: Analyze Span Breakdown
```
/api/orders (2000ms total)
├── authentication (50ms)
├── validate-request (20ms)
├── product-service (1500ms)      ← BOTTLENECK
│   ├── db-query (1400ms)         ← DB QUERY SLOW
│   └── transform (100ms)
├── inventory-service (100ms)
└── response-transform (30ms)
```

### Step 3: Dive Deeper
The product-service span shows 1500ms total, with db-query taking 1400ms - revealing the database as the bottleneck.

### Common Patterns:
1. **Single slow service**: One service dominates total time
2. **Cascading delays**: Slow service causes timeout retries
3. **Connection overhead**: Too many small HTTP calls
4. **Database bottleneck**: Slow queries within a service

---

## 26. How do you Correlate Logs with Traces?

### 1. Include Trace ID in Logs (MDC)
```java
@Component
public class TraceIdFilter extends HandlerInterceptorAdapter {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) {
        Span currentSpan = Span.current();
        MDC.put("traceId", currentSpan.getSpanContext().getTraceId());
        MDC.put("spanId", currentSpan.getSpanContext().getSpanId());
        return true;
    }
}
```

### 2. Structured Logging with Trace Context
```java
logger.info("Order processed", 
    Map.of(
        "traceId", MDC.get("traceId"),
        "orderId", order.getId(),
        "duration", durationMs
    )
);
```

### 3. Correlate in Logging Platform
Search by traceId to find all logs from a single request across all services.

---

## 27. What is Structured Logging?

**Structured logging** is a logging approach where log entries are formatted as structured data (typically JSON) rather than plain text.

### Plain vs Structured Logging:
```java
// Plain logging (BAD)
logger.info("User " + userId + " logged in from " + ipAddress);
// Output: User 123 logged in from 192.168.1.1

// Structured logging (GOOD)
logger.info("User login successful", 
    Map.of(
        "userId", userId,
        "ipAddress", ipAddress,
        "method", "POST"
    )
);
// Output: {"userId": 123, "ipAddress": "192.168.1.1", "method": "POST"}
```

### Benefits:
1. **Searchable**: Easy to filter by any field
2. **Parseable**: Can be queried programmatically
3. **Scalable**: Works well with log aggregation systems
4. **Consistent**: Standard format across services

---

## 28. How do you Implement Centralized Logging?

### Architecture:
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Service A  │  │  Service B  │  │  Service C  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        ▼
              ┌─────────────────────┐
              │   Log Aggregator   │
              │  (Fluentd/Logstash) │
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐
              │  Searchable Store  │
              │   (Elasticsearch)  │
              └──────────┬──────────┘
                         ▼
              ┌─────────────────────┐
              │     Visualization   │
              │      (Kibana)       │
              └─────────────────────┘
```

### Common Stack: ELK (Elasticsearch, Logstash, Kibana)

---

## 29. ELK vs OpenTelemetry

| Aspect | ELK Stack | OpenTelemetry |
|--------|-----------|---------------|
| **Primary Focus** | Logs | Traces, Metrics, Logs |
| **Components** | Elasticsearch, Logstash, Kibana | Collector, SDK, Language implementations |
| **Data Model** | Document-based logs | Unified data model for all signals |
| **Vendor Lock-in** | Limited | Vendor-neutral |
| **Setup Complexity** | Medium | High (initially) |
| **Use Case** | Log aggregation & analysis | Full-stack observability |

### When to Use:
- **ELK**: When logs are primary concern
- **OpenTelemetry**: For comprehensive observability across traces, metrics, and logs
- **Both**: Many organizations use both for their strengths

---

## 30. How do you Create Custom Micrometer Metrics?

### Counter
```java
@Service
public class OrderService {
    
    private final Counter orderCreatedCounter;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .tag("service", "order-service")
            .register(meterRegistry);
    }
    
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(request.toOrder());
        orderCreatedCounter.increment();
        return order;
    }
}
```

### Gauge
```java
@Service
public class CacheMonitorService {
    
    private final AtomicInteger cacheSize = new AtomicInteger(0);
    
    public CacheMonitorService(MeterRegistry meterRegistry) {
        Gauge.builder("cache.size", cacheSize, AtomicInteger::get)
            .description("Current cache size")
            .register(meterRegistry);
    }
}
```

### Timer
```java
@Service
public class PaymentService {
    
    private final Timer paymentTimer;
    
    public PaymentService(MeterRegistry meterRegistry) {
        this.paymentTimer = Timer.builder("payment.processing.time")
            .description("Time to process payment")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentTimer.record(() -> paymentGateway.charge(request));
    }
}
```

---

## 31. How do you Monitor Kafka Consumer Lag?

### Understanding Consumer Lag:
```
Producer → Topic → Partition → Consumer Lag = (Latest Offset) - (Consumer Position)
```

### Available Metrics:
```promql
kafka_consumer_records_lag_max{topic="orders",partition="0"}
kafka_consumer_fetch_manager_records_lag_max{topic="orders",partition="0"}
```

### Alert Rules:
```yaml
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_records_lag_max > 10000
  for: 5m
  labels:
    severity: warning
```

### Dashboard Queries:
```promql
# Consumer lag by topic and partition
sum by (topic, partition) (kafka_consumer_records_lag_max)
```

---

## 32. How do you Monitor Database Connection Pool Usage?

### HikariCP Metrics (Spring Boot)
```promql
hikaricp_connections_active{pool="HikariPool-1"}     # Active connections
hikaricp_connections_idle{pool="HikariPool-1"}        # Idle connections
hikaricp_connections{pool="HikariPool-1"}              # Total connections
hikaricp_connections_pending{pool="HikariPool-1"}     # Pending requests
hikaricp_connections_timeout_total{pool="HikariPool-1"} # Timeouts
```

### Alert Rules:
```yaml
- alert: HikariCPConnectionPoolExhausted
  expr: hikaricp_connections_active / hikaricp_connections >= 0.95
  for: 2m
  labels:
    severity: critical
```

### Dashboard Queries:
```promql
# Connection pool utilization percentage
100 * hikaricp_connections_active / hikaricp_connections
```

---

## 33. How do you Detect Memory Leaks Using Observability Tools?

### 1. JVM Heap Memory Trends
```promql
# Memory growth pattern
# Healthy: sawtooth (GC reclaiming memory)
# Leaky: continuous growth without GC reclaiming
jvm_memory_used_bytes{area="heap"}
```

### 2. GC Analysis
```promql
# GC pause frequency
rate(jvm_gc_pause_seconds_count[5m])

# Old generation collection frequency (sign of memory pressure)
rate(jvm_gc_pause_seconds_count{cause="Old GC"}[5m])
```

### 3. Leak Detection Patterns
```
Normal (Sawtooth):          Leaky (Climbing):
┌───┐                       ┌───┐
│   │    ┌───┐             │   │   ┌───┐
│   │    │   │    ┌───┐    │   │   │   │   ┌───┐
│   │    │   │    │   │    │   │   │   │   │   │
└───┘    └───┘    └───┘    └───┘   └───┘   └───┘
```

### Key Indicators:
- Continuous heap growth without GC reclaim
- Increasing GC frequency with Full GC
- OutOfMemoryError exceptions
- Increasing thread counts

---

## 34. How do you Detect High CPU Usage?

### JVM CPU Metrics
```promql
# CPU usage by JVM
process_cpu_usage           # Process CPU (0-1 scale)
system_cpu_usage            # System CPU (0-1 scale)

# By thread
jvm_threads_states_threads{state="runnable"}
```

### System Metrics
```promql
# System CPU
instance_cpu_usage{instance="app-server-1"}

# Per-core usage
instance_cpu_core_usage
```

### Alert Rules:
```yaml
- alert: HighCPUUsage
  expr: process_cpu_usage > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High CPU usage detected"
```

### Common Causes:
1. **Infinite loops** in code
2. **Excessive GC** (memory pressure)
3. **Regex operations** on large strings
4. **Crypto operations**
5. **Heavy computation** without caching

---

## 35. How do you Monitor a Spring Boot Application Running in Kubernetes?

### Architecture:
```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Pod 1     │  │   Pod 2     │  │   Pod 3     │    │
│  │ Spring Boot │  │ Spring Boot │  │ Spring Boot │    │
│  │  + Sidecar  │  │  + Sidecar  │  │  + Sidecar  │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         │                │                │            │
│  ┌──────┴────────────────┴────────────────┴──────┐    │
│  │              Service Monitor Layer              │    │
│  │         (Prometheus + node-exporter)          │    │
│  └───────────────────────┬────────────────────────┘    │
│                          │                             │
│                          ▼                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Monitoring Stack                   │   │
│  │   Prometheus  →  Grafana  →  Alertmanager      │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Kubernetes Monitoring Setup:
1. **Metrics Server**: kube-state-metrics, node-exporter
2. **Service Discovery**: Prometheus自动发现Kubernetes services
3. **Annotations**: prometheus.io/scrape: "true"
4. **Service Monitor**: For custom service discovery

### Key Metrics to Monitor:
```yaml
# Pod-level
kube_pod_container_resource_requests_cpu_cores
kube_pod_container_resource_limits_cpu_cores
kube_pod_status_phase

# Application-level
http_server_requests_seconds_count
jvm_memory_used_bytes
```

---

## 36. How do you Define Useful Alerts?

### Alert Design Principles:

### 1. **Actionable**
- Each alert should require action
- Avoid alerts that don't lead to action

### 2. **Relevant**
- Tied to business impact
- Meaningful to stakeholders

### 3. **Timely**
- Enough time to respond
- Not too early (noise) or too late (damage done)

### 4. **Prioritized**
- Critical, Warning, Info levels
- Different responses for different severities

### Alert Examples:
```yaml
# Good Alert - Actionable and Timely
- alert: HighErrorRate
  expr: |
    100 * sum(rate(http_requests_total{status=~"5.."}[5m])) / 
    sum(rate(http_requests_total[5m])) > 5
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 5%"
    description: "Service {{ $labels.service }} is experiencing high error rate"

# Bad Alert - Not Actionable
- alert: HighMemoryUsage
  expr: jvm_memory_used_bytes > 0
  for: 0m
  # Memory usage > 0 is always true!
```

---

## 37. What is an SLI?

**SLI (Service Level Indicator)** is a quantitative measure of service behavior.

### Common SLIs:
- **Availability**: % of time service is operational
- **Latency**: Response time distribution (p50, p95, p99)
- **Throughput**: Requests per second
- **Error Rate**: % of failed requests
- **Availability**: Uptime percentage

### Example:
```
SLI: API response time p95 < 500ms
SLI: Error rate < 1%
SLI: Availability > 99.9%
```

---

## 38. What is an SLO?

**SLO (Service Level Objective)** is a target value or range for an SLI.

### Example SLOs:
```yaml
SLOs:
  - name: API Availability
    sli: Availability
    target: 99.9%
    window: 30 days
  
  - name: API Latency
    sli: p95 response time
    target: < 500ms
    window: 30 days
  
  - name: Error Rate
    sli: 5xx error rate
    target: < 0.1%
    window: 30 days
```

### SLO vs SLI:
- **SLI**: The actual measured value
- **SLO**: The target you want to achieve

---

## 39. What is an SLA?

**SLA (Service Level Agreement)** is a contractual commitment between service provider and customer.

### SLA vs SLO:
| Aspect | SLA | SLO |
|--------|-----|-----|
| **Type** | Contract | Internal target |
| **Consequences** | Financial penalties | Internal goals |
| **Audience** | External customers | Internal teams |
| **Enforcement** | Legal binding | Operational discipline |

### Example:
```
SLA: 99.95% uptime (43.8 minutes downtime/month)
SLO: 99.9% uptime (internal target, higher than SLA)
SLI: Actual measured uptime (what we report)
```

---

## 40. How do you Design an End-to-End Observability Architecture for a Java Microservices System?

### Architecture Overview:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Java Microservices System                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Service │  │ Service │  │ Service │  │ Service │  │ Service │      │
│  │    A    │  │    B    │  │    C    │  │    D    │  │    E    │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │            │            │            │            │            │
│       └────────────┴────────────┴────────────┴────────────┘            │
│                                │                                        │
│                    ┌───────────┴───────────┐                           │
│                    │   OpenTelemetry SDK   │                           │
│                    │  (Micrometer + OTel)  │                           │
│                    └───────────┬───────────┘                           │
│                                │                                        │
└────────────────────────────────┼────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Observability Infrastructure                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    OpenTelemetry Collector                       │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐                   │   │
│  │  │ Traces    │  │ Metrics   │  │ Logs      │                   │   │
│  │  │ Receiver  │  │ Receiver  │  │ Receiver  │                   │   │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                   │   │
│  │        │              │              │                           │   │
│  │        └──────────────┼──────────────┘                           │   │
│  │                       │                                          │   │
│  │              ┌────────┴────────┐                                 │   │
│  │              │  Process & Route │                                 │   │
│  │              └────────┬────────┘                                 │   │
│  └───────────────────────┼─────────────────────────────────────────┘   │
│                          │                                              │
│          ┌───────────────┼───────────────┐                            │
│          │               │               │                            │
│          ▼               ▼               ▼                            │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │
│  │   Jaeger      │ │   Prometheus  │ │   Loki        │              │
│  │   (Traces)    │ │   (Metrics)   │ │   (Logs)      │              │
│  └───────────────┘ └───────────────┘ └───────────────┘              │
│          │               │               │                            │
│          └───────────────┴───────────────┘                            │
│                          │                                              │
│                          ▼                                              │
│                  ┌───────────────┐                                    │
│                  │   Grafana     │                                    │
│                  │   (Unified    │                                    │
│                  │   Dashboard)  │                                    │
│                  └───────────────┘                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Details:

### 1. **Application Layer**
```xml
<!-- Dependencies for each microservice -->
<dependencies>
    <!-- Micrometer for metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- OpenTelemetry -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
    </dependency>
    
    <!-- Logging with trace correlation -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
    </dependency>
</dependencies>
```

### 2. **Configuration (application.yml)**
```yaml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles: 0.5, 0.95, 0.99

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  service:
    name: ${spring.application.name}
```

### 3. **OpenTelemetry Collector (otel-collector-config.yaml)**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
  memory_limiter:
    check_interval: 1s
    limit_mib: 1000

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### 4. **Prometheus Configuration (prometheus.yml)**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']
  
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 5. **Grafana Dashboards**

#### Service Overview Dashboard:
- Request rate by service
- Error rate by service
- Latency (p50, p95, p99)
- Active instances

#### Infrastructure Dashboard:
- CPU usage
- Memory usage
- JVM metrics (heap, GC)
- Network I/O

#### Business Metrics Dashboard:
- Order creation rate
- Payment success rate
- User login attempts
- API usage by endpoint

### 6. **Alert Rules (alerting rules.yaml)**
```yaml
groups:
  - name: service-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          100 * sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / 
          sum(rate(http_server_requests_seconds_count[5m])) > 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          runbook_url: "https://wiki.runbook/high-error-rate"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
      
      - alert: HighConsumerLag
        expr: kafka_consumer_records_lag_max > 10000
        for: 5m
        labels:
          severity: warning
      
      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections >= 0.95
        for: 2m
        labels:
          severity: critical

  - name: infrastructure-alerts
    rules:
      - alert: HighCPU
        expr: process_cpu_usage > 0.8
        for: 5m
        labels:
          severity: warning
      
      - alert: HighMemoryUsage
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels:
          severity: warning
```

### 7. **SLO Monitoring**
```yaml
# slo-config.yaml
slos:
  - name: API Availability
    sli: availability
    target: 99.9
    window: 30d
    budget:
      percent: 5
      burn_rate:
        - threshold: 1
          percent: 50
        - threshold: 3
          percent: 50
  
  - name: API Latency p95
    sli: latency
    target: 99.5
    window: 30d
    threshold:
      value: 0.5s
```

### 8. **Logging Pipeline**
```yaml
# filebeat-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
      - add_trace_id
    output.elasticsearch:
      hosts: ["elasticsearch:9200"]
```

### Key Design Principles:

1. **Open Standards**: Use OpenTelemetry for vendor neutrality
2. **Unified Collection**: Single pipeline for all signals
3. **Correlation**: Include trace IDs in logs and metrics
4. **Alerting Strategy**: Actionable, prioritized alerts
5. **SLO-Based Monitoring**: Focus on business outcomes
6. **Retention**: Appropriate data retention policies
7. **Cost Efficiency**: Hot/warm/cold storage tiers
