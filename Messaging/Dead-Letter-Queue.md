# Dead Letter Queue (DLQ)

## What is DLQ?
A separate queue for messages that cannot be processed successfully after multiple retries.

## Purpose
- Prevent blocking the main queue
- Allow inspection of failed messages
- Enable separate handling and recovery

## Implementation

### 1. **Kafka DLQ**
```java
// Main topic: orders
// DLQ topic: orders.dlq

@KafkaListener(topics = "orders")
public void handle(ConsumerRecord<String, OrderEvent> record,
                  Acknowledgment ack) {
    try {
        orderService.process(record.value());
        ack.acknowledge();
    } catch (Exception ex) {
        // Send to DLQ
        deadLetterTemplate.send("orders.dlq", record);
    }
}
```

### 2. **RabbitMQ DLQ**
```java
// Configure DLQ in queue arguments
spring.rabbitmq.queue.name=orders
spring.rabbitmq.queue.arguments.x-dead-letter-exchange=dlx
spring.rabbitmq.queue.arguments.x-dead-letter-routing-key=orders.dlq
```

## DLQ Monitoring
- Alert on DLQ size
- Regular DLQ processing
- Message analysis and fix
- Replay successful messages

## Interview Tip
Explain that DLQs are a safety net, not a replacement for proper error handling. Always monitor and process DLQ messages.