# Event-Driven Audit Logging System

## Requirements
- Capture all financial transactions
- Immutable audit trail
- Real-time processing
- Queryable history
- Compliance reporting

## Architecture

### Event-Driven Design
```java
// Audit event
public class TransactionEvent {
    private Long transactionId;
    private String userId;
    private String action;
    private BigDecimal amount;
    private Instant timestamp;
    private Map<String, Object> metadata;
}

// Event publisher
@Service
public class AuditService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public void recordTransaction(Transaction transaction) {
        AuditEvent event = new AuditEvent(transaction);
        eventPublisher.publishEvent(event);
    }
}

// Event listener
@Component
public class AuditEventListener 
    implements ApplicationListener<AuditEvent> {
    
    @Async
    @Override
    public void onApplicationEvent(AuditEvent event) {
        auditRepository.save(event.getAudit());
        
        // Send to Kafka for real-time processing
        kafkaTemplate.send("audit-events", event);
    }
}
```

### Kafka Implementation
```java
// Kafka topic for audit events
 Topics: audit.transactions
   
// Event schema using Avro
{
  "type": "record",
  "name": "AuditEvent",
  "fields": [
    {"name": "transactionId", "type": "long"},
    {"name": "userId", "type": "string"},
    {"name": "action", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "timestamp", "type": "long"}
  ]
}
```

## Storage Strategy
- **Hot storage**: Elasticsearch for recent events
- **Cold storage**: S3/Glacier for long-term retention
- **Indexing**: Time-based, user-based, action-based

## Interview Tip
Explain that audit trails must be immutable and tamper-proof. Mention that event-driven architecture provides loose coupling and scalability.