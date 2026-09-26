# A Kafka Message Is Consumed Twice. How Do You Make the Consumer Idempotent?

## The Problem: At-Least-Once Delivery

Kafka guarantees **at-least-once** delivery by default. This means a message may be delivered to a consumer **more than once**. This can happen due to:

1. **Consumer crash** after processing but before committing the offset
2. **Rebalance** — consumer loses partition ownership
3. **Manual offset commit** after processing
4. **Retries** — consumer reprocesses the same message

```
Timeline:
1. Kafka sends message M to consumer
2. Consumer processes M (writes to DB)
3. Consumer crashes before committing offset
4. Consumer restarts, Kafka re-sends M (same offset)
5. Consumer processes M again — DUPLICATE!
```

## Solutions

### 1. Idempotent Consumer Pattern (Recommended)

Make the consumer's processing **idempotent** — processing the same message multiple times produces the same result.

#### Database-Level Idempotency

```java
@Service
public class OrderEventHandler {
    
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Use the event ID as a deduplication key
        String eventId = event.getEventId();
        
        // Check if this event was already processed
        if (eventProcessedRepository.existsByEventId(eventId)) {
            log.info("Event {} already processed — skipping", eventId);
            return;
        }
        
        // Process the event
        orderService.createOrder(event.getOrderDetails());
        
        // Record that this event was processed
        eventProcessedRepository.save(new ProcessedEvent(eventId));
    }
}
```

#### Database Upsert

```java
@Transactional
public void handleOrderCreated(OrderCreatedEvent event) {
    // Use INSERT ... ON CONFLICT (id) DO NOTHING
    // or MERGE INTO for databases that support it
    orderRepository.upsert(event.getOrder());
}
```

#### Unique Constraint

```sql
-- Ensure the order ID is unique
ALTER TABLE orders ADD CONSTRAINT uk_order_id UNIQUE (order_id);
```

### 2. Deduplication Table

Maintain a table of processed message IDs:

```sql
CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_processed_at ON processed_messages (processed_at);
```

```java
@Transactional
public void processMessage(KafkaMessage message) {
    // Check if already processed
    if (processedMessageRepository.existsById(message.getId())) {
        return;  // Skip — already processed
    }
    
    // Process the message
    doBusinessLogic(message);
    
    // Record that we processed it
    processedMessageRepository.save(new ProcessedMessage(message.getId()));
}
```

### 3. Idempotent Business Operations

Design business operations to be naturally idempotent:

```java
// Instead of:
public void updateBalance(String accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId);
    account.setBalance(account.getBalance().add(amount));  // Not idempotent!
    accountRepository.save(account);
}

// Use:
public void setBalance(String accountId, BigDecimal newBalance) {
    accountRepository.updateBalance(accountId, newBalance);  // Idempotent!
}

// Or use a version/sequence number:
public void updateBalance(String accountId, BigDecimal amount, long version) {
    int rows = accountRepository.updateBalanceWithVersion(
        accountId, amount, version);
    if (rows == 0) {
        // Version mismatch — someone else already updated
        // Re-read and retry, or skip
    }
}
```

### 4. Kafka Transactions (Exactly-Once Semantics)

Use Kafka's transactional API for exactly-once processing:

```java
// Producer with transactions
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("transactional.id", "my-transactional-id");
props.put("enable.idempotence", "true");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("output-topic", "key", "value"));
    producer.sendOffsetsToTransaction(consumerOffsets, "my-transactional-id");
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### 5. Consumer Offset Management

#### Manual Offset Commit (After Processing)

```java
@KafkaListener(topics = "orders")
public void processOrder(ConsumerRecord<String, String> record) {
    try {
        // Process the message
        processMessage(record.value());
        
        // Commit offset only after successful processing
        acknowledgment.acknowledge();
    } catch (Exception e) {
        // Don't commit — message will be re-consumed
        log.error("Failed to process message", e);
    }
}
```

#### At-Most-Once (Commit Before Processing)

```java
@KafkaListener(topics = "orders")
public void processOrder(ConsumerRecord<String, String> record, 
                         Acknowledgment acknowledgment) {
    // Commit first — at-most-once delivery
    acknowledgment.acknowledge();
    
    // Then process — if it fails, message is lost
    processMessage(record.value());
}
```

### 6. Deduplication with Redis

Use Redis as a fast deduplication store:

```java
@Service
public class IdempotentConsumer {
    private final RedisTemplate<String, String> redisTemplate;
    private static final String PROCESSED_KEY_PREFIX = "processed:";
    private static final int TTL_SECONDS = 3600;  // 1 hour

    public boolean isDuplicate(String messageId) {
        String key = PROCESSED_KEY_PREFIX + messageId;
        Boolean alreadyProcessed = redisTemplate.hasKey(key);
        if (alreadyProcessed) {
            return true;
        }
        // Set with TTL to prevent unbounded growth
        redisTemplate.opsForValue().set(key, "1", Duration.ofSeconds(TTL_SECONDS));
        return false;
    }

    public void processMessage(KafkaMessage message) {
        if (isDuplicate(message.getId())) {
            log.info("Duplicate message {} — skipping", message.getId());
            return;
        }
        doBusinessLogic(message);
    }
}
```

## Recommended Approach: Defense in Depth

Combine multiple strategies:

```java
@Service
public class RobustKafkaConsumer {
    
    @KafkaListener(topics = "orders")
    @Transactional
    public void processOrder(ConsumerRecord<String, String> record, 
                           Acknowledgment acknowledgment) {
        String messageId = record.key();
        
        // Layer 1: Check deduplication table
        if (processedMessageRepository.existsById(messageId)) {
            log.info("Duplicate message {} — skipping", messageId);
            acknowledgment.acknowledge();
            return;
        }
        
        try {
            // Layer 2: Process with idempotent business logic
            OrderCreatedEvent event = deserialize(record.value());
            orderService.createOrder(event);  // Uses upsert internally
            
            // Layer 3: Record processed message
            processedMessageRepository.save(new ProcessedMessage(messageId));
            
            // Commit offset
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Failed to process message {}", messageId, e);
            // Don't acknowledge — message will be re-consumed
        }
    }
}
```

## Key Takeaway

> To make Kafka consumers idempotent, use a **deduplication table** or **idempotent business operations** as the primary defense, combined with **manual offset commits** (after processing) and optionally **Kafka transactions** for exactly-once semantics. The key is to ensure that processing the same message multiple times produces the same result as processing it once.
