# Message Ordering in Kafka

## Challenges
- Partition-level ordering only
- Cross-partition ordering not guaranteed
- Reordering possible with retries

## Strategies

### 1. **Partition Key**
```java
// Same key → same partition → ordered
producer.send(new ProducerRecord<>(topic, orderId, event));
```

### 2. **Single Partition**
- Guaranteed ordering
- Limited scalability

### 3. **In-Order Processing**
```java
// Process messages in order within partition
@KafkaListener(topics = "orders", concurrency = 1)
public void handle(OrderEvent event) {
    // Process sequentially
    orderService.process(event);
}
```

### 4. **Sequence Numbers**
```java
// Add sequence number to detect out-of-order
class OrderedEvent {
    Long orderId;
    Long sequenceNumber;
    OrderEvent event;
}

// Store last processed sequence
Map<Long, Long> lastSequence = new ConcurrentHashMap<>();

public void process(OrderedEvent orderedEvent) {
    Long lastSeq = lastSequence.get(orderedEvent.orderId);
    if (orderedEvent.sequenceNumber <= lastSeq) {
        // Out of order or duplicate
        return;
    }
    
    // Process in order
    lastSequence.put(orderedEvent.orderId, orderedEvent.sequenceNumber);
}
```

## Interview Tip
Explain that Kafka guarantees ordering within a partition, not across partitions. Design partition keys carefully for ordering requirements.