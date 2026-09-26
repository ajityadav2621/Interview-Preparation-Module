# How Does Spring Boot Detect and Configure Embedded Tomcat?

## Detection Process

### 1. **Classpath Detection**
- Spring Boot checks for Tomcat dependency
- `spring-boot-starter-web` includes `tomcat-embed-core`
- Auto-configuration detects `Servlet` classes

### 2. **Auto-Configuration**
```java
@Configuration
@ConditionalOnClass(Servlet.class)
public class EmbeddedServletContainerAutoConfiguration {
    
    @Configuration
    @ConditionalOnClass(TomcatServletWebServerFactory.class)
    public static class TomcatWebServerFactoryConfiguration {
        
        @Bean
        @ConditionalOnMissingBean
        public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
            return new TomcatServletWebServerFactory();
        }
    }
}
```

### 3. **Configuration Properties**
```yaml
server:
  port: 8080
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    connection-timeout: 20000
```

### 4. **Server Startup**
- `TomcatServletWebServerFactory` creates Tomcat instance
- Configures connectors, context, and servlets
- Starts Tomcat on configured port

## Interview Tip
Explain that Spring Boot uses auto-configuration to detect embedded servers. You can customize through `application.yml` or by defining custom beans.