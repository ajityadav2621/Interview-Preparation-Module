# Micrometer

## What is Micrometer?
A metrics library that provides a unified API for JVM and non-JVM metrics.

## Key Features
- **Unified API**: Works with different monitoring systems
- **Auto-configuration**: Spring Boot integration
- **Tagging**: Dimensional metrics
- **Custom metrics**: Application-specific measurements

## Integration
```java
dependencies {
    implementation 'io.micrometer:micrometer-registry-prometheus'
}

// Spring Boot auto-configures Micrometer
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## Custom Metrics
```java
@Component
public class CustomMetrics {
    
    private final MeterRegistry registry;
    private final Counter orderCounter;
    
    public CustomMetrics(MeterRegistry registry) {
        this.registry = registry;
        this.orderCounter = Counter.builder("orders.total")
            .tag("type", "created")
            .register(registry);
    }
    
    public void recordOrder() {
        orderCounter.increment();
    }
}
```

## Interview Tip
Explain that Micrometer standardizes metrics collection across different backends. It's the foundation for Spring Boot Actuator metrics.