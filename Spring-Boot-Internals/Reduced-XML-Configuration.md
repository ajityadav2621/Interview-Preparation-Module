# How Spring Boot Reduces XML Configuration

## Annotation-Based Configuration

### 1. **Component Scanning**
```java
// Instead of XML bean definitions
@Component
public class UserService {
    // Bean automatically registered
}

// XML equivalent (what Spring Boot eliminates):
<bean id="userService" class="com.example.UserService"/>
```

### 2. **Java Configuration**
```java
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public JpaTransactionManager transactionManager() {
        return new JpaTransactionManager();
    }
}
```

### 3. **Auto-Configuration**
- Spring Boot auto-configures common beans
- Only requires explicit configuration for custom cases
- Reduces configuration to properties

## Benefits
- **Type Safety**: Compile-time checking
- **IDE Support**: Refactoring and autocomplete
- **Testability**: Easier to test configuration
- **Maintainability**: Less boilerplate code

## Interview Tip
Explain that Spring Boot doesn't eliminate configuration - it moves from XML to Java annotations and properties. The goal is to reduce boilerplate.