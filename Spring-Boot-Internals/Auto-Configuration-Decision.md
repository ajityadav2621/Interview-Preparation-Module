# How Does Spring Boot Decide Which Auto-Configuration to Apply?

## Auto-Configuration Mechanism

### 1. **Auto-Configuration Detection**
- Spring Boot scans `@EnableAutoConfiguration` (or `@SpringBootApplication`)
- Uses `SpringFactoriesLoader` to find auto-configuration classes
- Reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

### 2. **Conditional Evaluation**
Each auto-configuration class uses `@Conditional` annotations:
- `@ConditionalOnMissingBean`: Only if bean not defined
- `@ConditionalOnProperty`: Based on configuration properties
- `@ConditionalOnClass`: Only if class is on classpath
- `@ConditionalOnMissingClass`: Only if class not on classpath

### Example
```java
@Configuration
@ConditionalOnProperty(name = "server.port")
public class ServerAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public EmbeddedServletContainer embeddedServletContainer() {
        return new TomcatEmbeddedServletContainer();
    }
}
```

### 3. **Auto-Configuration Order**
- Ordered by `@AutoConfigureOrder` or `@Order`
- Higher priority configurations apply first
- User-defined beans override auto-configured ones

## Interview Tip
Explain that auto-configuration is "opinionated" - Spring Boot provides defaults but allows customization through user-defined beans.