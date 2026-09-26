# Duplicate Message Handling

## Causes
- At-least-once delivery semantics
- Network issues causing acknowledgments to be lost
- Consumer retries

## Solutions

### 1. **Idempotent Consumers**
```java
// Make processing idempotent
@Component
public class OrderConsumer {
    
    @KafkaListener(topics = "orders")
    public void handle(OrderEvent event) {
        // Check if already processed
        if (orderService.isProcessed(event.getOrderId())) {
            return; // Skip duplicate
        }
        
        // Process idempotently
        orderService.process(event);
    }
}
```

### 2. **Message Deduplication**
```java
// Store processed message IDs
Set<String> processedMessages = ConcurrentHashMap.newKeySet();

public void process(Message message) {
    String messageId = message.getId();
    
    if (!processedMessages.add(messageId)) {
        // Duplicate detected
        return;
    }
    
    // Process message
    handle(message);
}
```

### 3. **Database Constraints**
```sql
CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT NOW()
);
```

## Interview Tip
Explain that Kafka provides at-least-once delivery by default. Idempotency must be implemented at the consumer level.