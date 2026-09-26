# Correlating Logs with Traces

## Structured Logging with Trace Context
```java
@Component
public class TraceIdConverter {
    
    public String traceId() {
        Span currentSpan = Tracer.currentSpan();
        if (currentSpan != null) {
            return currentSpan.getSpanContext().traceId();
        }
        return null;
    }
}
```

## Logback Configuration
```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %X{traceId} %msg%n</pattern>
        </encoder>
    </appender>
</configuration>
```

## Benefits
- Search logs by trace ID
- Correlate logs with spans
- Faster debugging
- Complete request context

## Interview Tip
Explain that correlating logs with traces provides complete request context, making debugging much faster in distributed systems.