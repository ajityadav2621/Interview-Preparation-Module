# Kafka Message Processing with Database Rollback

## Problem Scenario
- Consume Kafka message (XML)
- Transform and process
- Persist to database
- Publish to downstream Kafka topic
- **Issue**: Database updated but downstream message not published due to error

## Solution: Transactional Outbox Pattern

### 1. **Outbox Table Approach**
```java
@Entity
@Table(name = "outbox")
public class OutboxMessage {
    
    @Id
    private String id;
    
    private String payload;
    private String topic;
    private String status; // PENDING, SENT, FAILED
    private Instant createdAt;
    private Instant updatedAt;
    
    // Getters and setters
}
```

### 2. **Atomic Operations**
```java
@Service
public class MessageProcessingService {
    
    private final EntityManager entityManager;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Transactional
    public void processMessage(String kafkaMessage) {
        try {
            // 1. Parse and transform message
            DataObject data = parseAndTransform(kafkaMessage);
            
            // 2. Save to database
            DataEntity entity = saveToDatabase(data);
            
            // 3. Create outbox entry
            OutboxMessage outbox = new OutboxMessage();
            outbox.setPayload(transformToDownstreamFormat(data));
            outbox.setTopic("downstream-topic");
            outbox.setStatus("PENDING");
            entityManager.persist(outbox);
            
            // 4. Commit transaction atomically
            // Both database and outbox are committed together
            
        } catch (Exception ex) {
            // Transaction will rollback automatically
            throw new RuntimeException("Processing failed", ex);
        }
    }
}
```

### 3. **Outbox Relay Service**
```java
@Component
public class OutboxRelay {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 5000)
    public void relayMessages() {
        List<OutboxMessage> pendingMessages = 
            outboxRepository.findByStatus("PENDING");
        
        for (OutboxMessage message : pendingMessages) {
            try {
                // Send to Kafka
                kafkaTemplate.send(
                    message.getTopic(),
                    message.getPayload()
                ).get();
                
                // Mark as sent
                message.setStatus("SENT");
                outboxRepository.save(message);
                
            } catch (Exception ex) {
                // Mark as failed for retry
                message.setStatus("FAILED");
                message.setRetryCount(message.getRetryCount() + 1);
                outboxRepository.save(message);
            }
        }
    }
}
```

### 4. **Alternative: Two-Phase Commit (2PC)**
```java
// Not recommended for microservices due to complexity
@TwoPhaseCommit
public class DistributedTransactionManager {
    
    @Prepare
    public boolean prepare(DatabaseOperation dbOp, KafkaOperation kafkaOp) {
        // Prepare both resources
        return dbOp.prepare() && kafkaOp.prepare();
    }
    
    @Commit
    public void commit(DatabaseOperation dbOp, KafkaOperation kafkaOp) {
        dbOp.commit();
        kafkaOp.commit();
    }
}
```

## Interview Tip
Explain that the outbox pattern ensures consistency between database and message queue. It's a common pattern in microservices architectures for handling distributed transactions.