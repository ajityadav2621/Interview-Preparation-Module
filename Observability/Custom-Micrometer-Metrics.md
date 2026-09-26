# Custom Micrometer Metrics

## Counter
```java
@Component
public class OrderMetrics {
    private final Counter orderCounter;
    
    public OrderMetrics(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.total")
            .tag("type", "created")
            .register(registry);
    }
    
    public void recordOrder() {
        orderCounter.increment();
    }
}
```

## Timer
```java
private final Timer processingTimer;

public OrderMetrics(MeterRegistry registry) {
    this.processingTimer = Timer.builder("orders.processing.time")
        .register(registry);
}

public void recordProcessingTime(Runnable task) {
    processingTimer.record(task);
}
```

## Gauge
```java
private final Gauge activeOrdersGauge;

public OrderMetrics(MeterRegistry registry, OrderService orderService) {
    this.activeOrdersGauge = Gauge.builder("orders.active")
        .register(registry, orderService, 
            service -> service.getActiveOrderCount());
}
```

## Distribution Summary
```java
private final DistributionSummary orderAmountSummary;

public OrderMetrics(MeterRegistry registry) {
    this.orderAmountSummary = DistributionSummary.builder("orders.amount")
        .register(registry);
}

public void recordOrderAmount(BigDecimal amount) {
    orderAmountSummary.record(amount.doubleValue());
}
```

## Interview Tip
Explain that Micrometer provides different metric types for different use cases: Counter for rates, Timer for durations, Gauge for current values.