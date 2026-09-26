# Asynchronous Message Consumption by 5 Microservices

## Requirements
- One message source
- 5 different microservices need to read messages asynchronously
- Each service processes independently

## Approaches

### 1. **Kafka Topic with Multiple Consumer Groups**
```java
// Service A - Consumer Group 1
@KafkaListener(topics = "shared-topic", groupId = "service-a-group")
public void consumeByServiceA(String message) {
    // Process for Service A
}

// Service B - Consumer Group 2
@KafkaListener(topics = "shared-topic", groupId = "service-b-group")
public void consumeByServiceB(String message) {
    // Process for Service B
}

// Service C - Consumer Group 3
@KafkaListener(topics = "shared-topic", groupId = "service-c-group")
public void consumeByServiceC(String message) {
    // Process for Service C
}
```

### Configuration
```yaml
spring:
  kafka:
    consumer:
      group-id: default-group
    listener:
      concurrency: 3
```

### 2. **Kafka Topic with Partitioning**
```
Topic: order-events (5 partitions)
  Partition 0: Service A reads
  Partition 1: Service B reads
  Partition 2: Service C reads
  Partition 3: Service D reads
  Partition 4: Service E reads
```

### 3. **Message Queue with Multiple Queues**
```java
// Publisher sends to fanout exchange
rabbitTemplate.convertAndSend("orders-exchange", "", message);

// Each service has its own queue
@RabbitListener(queues = "service-a-queue")
public void consumeA(String message) { }

@RabbitListener(queues = "service-b-queue")
public void consumeB(String message) { }
```

### 4. **Event Bus Pattern**
```java
// Publish event
eventBus.post(new OrderCreatedEvent(order));

// Each service listens
@Component
public class ServiceAListener {
    @Subscribe
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process for Service A
    }
}

@Component
public class ServiceBListener {
    @Subscribe
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process for Service B
    }
}
```

## Why Kafka Consumer Groups?

### Advantages
- **Independent processing**: Each service processes all messages
- **Scalability**: Each service can scale independently
- **Fault tolerance**: One service failure doesn't affect others
- **Load balancing**: Consumers within group share load

### Configuration
```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "default-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(config);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

## Interview Tip
Explain that Kafka consumer groups are the ideal solution for this scenario. Each microservice should have its own consumer group to independently process all messages.