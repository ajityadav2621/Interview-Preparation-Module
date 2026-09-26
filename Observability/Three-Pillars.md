# Three Pillars of Observability

## 1. **Metrics**
- **Definition**: Numerical measurements over time
- **Examples**: CPU usage, request count, latency
- **Use Case**: Trending, alerting, dashboards
- **Tools**: Prometheus, Graphite, InfluxDB

## 2. **Logs**
- **Definition**: Timestamped events with context
- **Examples**: Error messages, application events
- **Use Case**: Debugging, audit trails
- **Tools**: ELK Stack, Splunk, Fluentd

## 3. **Traces**
- **Definition**: Request flows through distributed systems
- **Examples**: End-to-end request paths, service interactions
- **Use Case**: Performance analysis, bottleneck identification
- **Tools**: Jaeger, Zipkin, OpenTelemetry

## Why All Three?
- **Metrics**: High-level health monitoring
- **Logs**: Detailed event investigation
- **Traces**: Distributed system debugging

## Interview Tip
Explain that each pillar serves different purposes. Effective observability requires all three working together.