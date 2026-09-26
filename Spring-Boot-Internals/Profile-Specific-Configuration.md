# Profile-Specific Configuration in Spring Boot

## Profile Support

### 1. **Profile Properties Files**
```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# Development profile
spring:
  config:
    activate:
      profile: dev

server:
  port: 8080

logging:
  level:
    com.example: DEBUG

---
# Production profile  
spring:
  config:
    activate:
      profile: prod

server:
  port: 8080

logging:
  level:
    com.example: INFO

database:
  url: jdbc:mysql://prod-db:3306/app
```

### 2. **Programmatic Profile Configuration**
```java
@Component
public class ProfileConfiguration implements CommandLineRunner {
    
    @Value("${spring.profiles.active}")
    private String activeProfile;
    
    @Override
    public void run(String... args) throws Exception {
        System.out.println("Active profile: " + activeProfile);
        
        if ("prod".equals(activeProfile)) {
            // Production-specific initialization
            enableProductionFeatures();
        }
    }
}
```

### 3. **Profile-Specific Beans**
```java
@Configuration
@Profile("dev")
public class DevConfiguration {
    
    @Bean
    public DataSource devDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:h2:mem:test")
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfiguration {
    
    @Bean
    public DataSource prodDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://prod-db:3306/app")
            .build();
    }
}
```

## Interview Tip
Explain that profiles allow different configurations for different environments. Use them for database URLs, logging levels, and feature flags.