# Kafka Consumer Crash While Processing

## What Happens

### 1. **If Crash Before Commit**
- Consumer restarts and reprocesses message
- Potential duplicate processing

### 2. **If Crash After Processing But Before Commit**
- Message reprocessed on restart
- Need idempotency to handle duplicates

## Solutions

### 1. **Auto-Commit with Idempotency**
```java
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) {
    // Process idempotently
    orderService.process(event);
    
    // Auto-commit happens periodically
}
```

### 2. **Manual Commit with Idempotency**
```java
@KafkaListener(topics = "orders")
public void handle(ConsumerRecord<String, OrderEvent> record,
                  Acknowledgment ack) {
    try {
        // Process idempotently
        orderService.process(record.value());
        
        // Commit after successful processing
        ack.acknowledge();
    } catch (Exception ex) {
        // Don't commit - will be reprocessed
        throw ex;
    }
}
```

### 3. **Transaction ID**
```java
// Use transactional ID for exactly-once semantics
spring.kafka.producer.transaction-id-prefix=tx-
```

## Interview Tip
Explain that Kafka default is at-least-once delivery. Exactly-once requires idempotency or transactions.