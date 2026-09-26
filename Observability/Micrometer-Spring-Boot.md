# Spring Boot and Micrometer Integration

## Auto-configuration
- Spring Boot Actuator auto-configures Micrometer
- Provides default metrics (JVM, system, HTTP)
- Exposes metrics via HTTP endpoints

## Configuration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

## Default Metrics
- JVM: Heap, threads, classes
- System: CPU, memory, processor
- HTTP: Requests, latency, errors
- Process: Uptime, CPU usage

## Interview Tip
Explain that Micrometer + Spring Boot Actuator provides production-ready metrics out of the box.