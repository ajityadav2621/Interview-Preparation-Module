# Spring Boot Application Startup Flow

## Step-by-Step Process

### 1. **Application Entry Point**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2. **SpringApplication.run() Execution**
1. **Initialize SpringerApplication** instance
2. **Determine Web Application Type** (SERVLET, REACTIVE, NONE)
3. **Create Bootstrap Context** (early initialization)
4. **Load Application Listeners** (SpringApplicationRunListeners)
5. **Prepare Environment** (properties, profiles)
6. **Create Context** (AnnotationConfigApplicationContext)
7. **Refresh Context** (bean creation and dependency injection)
8. **After Refresh** (post-processing, embedded server start)

### 3. **Bean Creation Order**
1. Configuration classes processed
2. Auto-configuration applied
3. Component scanning (`@Component`, `@Service`, `@Repository`)
4. Bean dependencies resolved
5. Bean post-processors applied
6. Lifecycle methods called (`@PostConstruct`)

### 4. **Embedded Server Start**
- Tomcat/Jetty starts on configured port
- Application ready to serve requests

## Interview Tip
Explain that Spring Boot startup involves multiple phases: environment preparation, context creation, bean initialization, and server startup. Each phase can be customized.