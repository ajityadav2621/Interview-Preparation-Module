# Missing Logs During Incident - What to Do?

## Impact
- Cannot determine root cause
- Difficult to reconstruct timeline
- Limited forensic analysis

## What to Do

### 1. **Check Logging Configuration**
```java
// Verify logging is enabled
logging:
  level:
    com.example: DEBUG
  file:
    name: /var/log/application.log
```

### 2. **Check Log Storage**
- Disk space
- Log rotation
- Centralized logging health
- Retention policies

### 3. **Alternative Data Sources**
- Application metrics
- APM traces
- Database audit logs
- Network logs
- Infrastructure logs

### 4. **Improve Logging**
```java
// Structured logging with correlation IDs
@Component
public class LoggingAspect {
    
    @Around("execution(* com.example..*(..))")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String traceId = MDC.get("traceId");
        long start = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - start;
            
            log.info("Method {} completed in {}ms", 
                    joinPoint.getSignature().getName(), duration);
            
            return result;
        } catch (Exception ex) {
            long duration = System.currentTimeMillis() - start;
            log.error("Method {} failed in {}ms", 
                     joinPoint.getSignature().getName(), duration, ex);
            throw ex;
        }
    }
}
```

## Interview Tip
Explain that missing logs are a serious problem. Implement structured logging with correlation IDs and ensure logs are stored redundantly.