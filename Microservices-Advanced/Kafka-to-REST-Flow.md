# Microservice B Consumes Kafka and Posts to REST API

## Flow Architecture

### 1. **Kafka Consumption**
```java
@KafkaListener(topics = "service-b-topic")
public void consumeMessage(String message) {
    // Process message
    processAndForward(message);
}
```

### 2. **Processing and Forwarding**
```java
@Service
public class MessageProcessor {
    
    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;
    
    public MessageProcessor(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        this.objectMapper = new ObjectMapper();
    }
    
    public void processAndForward(String kafkaMessage) {
        try {
            // Parse message
            DataObject data = objectMapper.readValue(kafkaMessage, DataObject.class);
            
            // Transform if needed
            DataObject transformed = transform(data);
            
            // Forward to REST API
            ResponseEntity<Response> response = restTemplate.postForEntity(
                "http://service-c/api/endpoint",
                transformed,
                Response.class
            );
            
            // Handle response
            if (response.getStatusCode().is2xxSuccessful()) {
                log.info("Message forwarded successfully");
            } else {
                log.error("Failed to forward message: {}", response.getStatusCode());
            }
            
        } catch (Exception ex) {
            log.error("Error processing message", ex);
            // Handle error appropriately
        }
    }
}
```

### 3. **Configuration**
```java
@Configuration
public class KafkaConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        // Configure HTTP message converters
        List<HttpMessageConverter<?>> converters = new ArrayList<>();
        converters.add(new MappingJackson2HttpMessageConverter());
        template.setInterceptors(converters);
        return template;
    }
}
```

## Interview Tip
Explain that this is a typical integration pattern: Kafka → processing → REST. Use proper error handling and idempotency for reliability.