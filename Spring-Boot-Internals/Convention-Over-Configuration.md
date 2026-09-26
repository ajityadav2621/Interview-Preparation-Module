# Why Does Spring Boot Prefer Convention Over Configuration?

## Convention Over Configuration (CoC)

### Definition
- Spring Boot makes opinions about how things should be done
- Follows standard conventions
- Only requires configuration when deviating from defaults

### Benefits

#### 1. **Faster Development**
- No XML configuration needed
- Sensible defaults provided
- Focus on business logic

#### 2. **Consistency**
- Standardized project structure
- Predictable behavior
- Easier onboarding

#### 3. **Reduced Configuration**
```java
// Before Spring Boot
<servlet>
    <servlet-name>app</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>

// After Spring Boot - Just add @SpringBootApplication
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### When to Override
- Non-standard database ports
- Custom thread pool sizes
- Specialized caching requirements
- External service URLs

## Interview Tip
Explain that CoC reduces boilerplate but doesn't eliminate configuration. Spring Boot provides extension points when defaults aren't appropriate.