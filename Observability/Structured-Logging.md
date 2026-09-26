# Structured Logging

## What is Structured Logging?
Logging format that includes key-value pairs for easier machine parsing and analysis.

## Benefits
- **Queryable**: Easy to search and analyze
- **Consistent**: Standardized format across services
- **Rich context**: Includes metadata and correlation IDs
- **Tool integration**: Works with log analysis tools

## Implementation
```java
// Using MDC (Mapped Diagnostic Context)
MDC.put("traceId", traceId);
MDC.put("userId", userId);
MDC.put("orderId", orderId);

log.info("Order created: {}", orderId);

// Structured format
log.info("order.created {}", Map.of(
    "orderId", orderId,
    "userId", userId,
    "amount", amount
));
```

## Logback Configuration
```xml
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %X{traceId} %msg%n</pattern>
```

## Interview Tip
Explain that structured logging makes logs machine-readable and enables better analysis. Use correlation IDs for request tracking.