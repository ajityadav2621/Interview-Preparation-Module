# Externalized Configuration in Spring Boot

## Configuration Sources

### 1. **Property File Hierarchy**
1. `application.yml` or `application.properties`
2. Profile-specific: `application-{profile}.yml`
3. External config files (outside JAR)
4. Command line arguments
5. Environment variables
6. System properties
7. @Value annotations
8. @ConfigurationProperties classes

### 2. **Configuration Properties**
```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    
    private String name;
    private int timeout;
    private List<String> servers = new ArrayList<>();
    
    // Getters and setters
}
```

### 3. **Usage in Components**
```java
@Component
public class AppConfig {
    
    @Value("${server.port}")
    private int port;
    
    @Value("${spring.profiles.active:dev}")
    private String activeProfile;
    
    private final AppProperties appProperties;
    
    public AppConfig(AppProperties appProperties) {
        this.appProperties = appProperties;
    }
}
```

### 4. **YAML Configuration**
```yaml
app:
  name: My Application
  timeout: 30000
  servers:
    - server1.example.com
    - server2.example.com
```

## Interview Tip
Explain that externalized configuration allows the same code to run in different environments without code changes. Use @ConfigurationProperties for complex configuration.