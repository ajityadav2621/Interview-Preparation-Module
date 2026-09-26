# SpringFactoriesLoader - What Is It?

## Purpose
SpringFactoriesLoader loads factory implementations from `META-INF/spring.factories` files.

## How It Works

### 1. **File Location**
- `META-INF/spring.factories` in JAR files
- Contains factory interface → implementation mappings

### 2. **Example Content**
```properties
# Auto-configuration classes
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.webEmbeddedServletContainerAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.JpaAutoConfiguration

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.context.logging.LoggingApplicationListener

# Failure Analysis
org.springframework.boot.bootreactivewebapplicationlistener=\
org.springframework.boot.bootreactivewebapplicationlistener
```

### 3. **Usage in Spring Boot**
```java
// Load auto-configuration classes
List<String> factories = SpringFactoriesLoader
    .loadFactoryNames(EnableAutoConfiguration.class, classLoader);

// Process each factory
for (String factory : factories) {
    Class<?> factoryClass = Class.forName(factory);
    // Instantiate and process
}
```

## Interview Tip
Explain that SpringFactoriesLoader is the mechanism behind Spring Boot's auto-configuration. It allows JARs to register auto-configuration without explicit imports.