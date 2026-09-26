# Configuring Health Checks in Spring Boot

## Built-in Health Indicators

### 1. **Auto-configured Health Checks**
- **DbHealthIndicator**: Database connectivity
- **DiskSpaceHealthIndicator**: Disk space
- **MongoHealthIndicator**: MongoDB
- **RedisHealthIndicator**: Redis
- **CassandraHealthIndicator**: Cassandra

### 2. **Accessing Health Info**
```java
// Endpoint
GET /actuator/health

// Response
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "hello": 1
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 1000000000,
        "free": 500000000
      }
    }
  }
}
```

### 3. **Custom Health Indicator**
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // Check custom service
        boolean isServiceAvailable = checkService();
        
        if (isServiceAvailable) {
            return Health.up()
                .build();
        } else {
            return Health.down()
                .withDetail("error", "Service unavailable")
                .build();
        }
    }
    
    private boolean checkService() {
        // Implement health check logic
        return true;
    }
}
```

## Interview Tip
Explain that health checks are critical for orchestration systems like Kubernetes. Custom indicators allow application-specific health validation.