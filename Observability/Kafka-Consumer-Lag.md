# Monitoring Kafka Consumer Lag

## What is Consumer Lag?
The difference between the latest message offset and the consumer's current position.

## Key Metrics
- **Consumer lag**: Messages behind
- **Consumer rate**: Messages processed per second
- **Partition lag**: Per-partition lag

## Monitoring Implementation
```java
@Component
public class KafkaMetrics {
    private final MeterRegistry registry;
    
    public KafkaMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    public void recordLag(String consumerGroup, long lag) {
        Gauge.builder("kafka.consumer.lag")
            .tag("group", consumerGroup)
            .register(registry)
            .value(lag);
    }
}
```

## Prometheus Query
```promql
kafka_consumer_lag{group="order-consumer"} > 1000
```

## Interview Tip
Explain that consumer lag indicates how far behind a consumer is. High lag can indicate processing bottlenecks or consumer issues.