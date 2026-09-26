# What Happens When You Add spring-boot-starter-web?

## Dependency Resolution

### 1. **Transitive Dependencies**
- Spring Boot parent POM manages versions
- Starter includes all required dependencies:
  - spring-web
  - spring-boot-autoconfigure
  - jackson-databind
  - tomcat-embed
  - spring-context

### 2. **Embedded Server Configuration**
- Auto-configures embedded Tomcat (or Jetty/Undertow)
- Sets default port (8080)
- Configures servlet container properties

### 3. **Spring MVC Auto-Configuration**
- Detects `@RestController`, `@Controller`
- Configures `DispatcherServlet`
- Sets up message converters (JSON, XML)
- Configures static resource handling

## Example
```java
// Just add dependency, no XML needed
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}

// Controller works immediately
@RestController
public class HelloController {
    
    @GetMapping("/hello")
    public String hello() {
        return "Hello World";
    }
}
```

## Interview Tip
Explain that starters are "convenience" dependencies that include everything needed for specific functionality. They use Spring Boot's dependency management to avoid version conflicts.