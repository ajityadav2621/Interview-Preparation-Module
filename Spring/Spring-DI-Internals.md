# Spring Boot Dependency Injection Internally

## How DI Works

### 1. **Bean Definition**
- @Component, @Service, @Repository, @Configuration
- @Bean methods in configuration classes
- Auto-configuration from starters
- Component scanning

### 2. **Bean Lifecycle**
```java
// 1. Instantiation
bean = beanClass.newInstance();

// 2. Populate properties (dependency injection)
bean.setProperty(value);

// 3. Aware interfaces
if (bean instanceof BeanNameAware) {
    ((BeanNameAware) bean).setBeanName(id);
}

// 4. BeanPostProcessor (before init)
bean = postProcessBeforeInitialization(bean, name);

// 5. @PostConstruct
@PostConstruct
public void init() { /* ... */ }

// 6. BeanPostProcessor (after init)
bean = postProcessAfterInitialization(bean, name);

// 7. Destroy callbacks
@PreDestroy
public void destroy() { /* ... */ }
```

### 3. **Proxy Creation**
- @Transactional creates proxies
- @Cacheable creates proxies
- AOP aspects create proxies
- CGLIB or JDK dynamic proxies

## Interview Tip
Explain that Spring uses reflection for DI and proxies for transaction management. Mention that circular dependencies are handled through三级缓存 (三级缓存).