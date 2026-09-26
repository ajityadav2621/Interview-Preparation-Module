# Poison Messages in Kafka

## What are Poison Messages?
Messages that consistently fail processing, causing infinite retries and blocking the consumer.

## Causes
- Malformed data
- Business logic errors
- External service failures
- Schema evolution issues

## Solutions

### 1. **Dead Letter Queue (DLQ)**
```java
@KafkaListener(topics = "orders", 
               containerFactory = "retryContainerFactory")
public void handle(OrderEvent event) {
    try {
        orderService.process(event);
    } catch (Exception ex) {
        // Send to DLQ after max retries
        throw new MessageProcessingException("Failed to process", ex);
    }
}

// Configure DLQ
 topics: orders.dlq
```

### 2. **Retry with Backoff**
```java
@Retryable(value = {Exception.class},
           maxAttempts = 3,
           backoff = @Backoff(delay = 2000))
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) {
    orderService.process(event);
}
```

### 3. **Poison Message Handling**
```java
@KafkaListener(topics = "orders")
public void handle(ConsumerRecord<String, OrderEvent> record) {
    try {
        orderService.process(record.value());
    } catch (PoisonMessageException ex) {
        // Send to DLQ or skip
        deadLetterTemplate.send("orders.dlq", record);
    }
}
```

## Interview Tip
Explain that DLQs are essential for handling poison messages. Monitor DLQ size and have alerting in place.