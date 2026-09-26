# Spring JPA Internally Uses SessionFactory or EntityManager?

## Spring Data JPA Architecture

### Internal Components
- **JpaEntityManagerFactory**: Creates EntityManagerFactory
- **EntityManagerFactory**: Creates EntityManagers (one per transaction)
- **EntityManager**: JPA interface for entity operations
- **Hibernate Session**: Implementation of EntityManager

### Spring Data JPA Uses Both
```java
// Spring Boot Auto-configuration
@Configuration
@ConditionalOnClass({EntityManager.class, Hibernate.class})
public class JpaAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource dataSource) {
        
        HibernateJpaVendor vendor = new HibernateJpaVendor();
        // Hibernate is used as JPA provider
        // SessionFactory is created internally by Hibernate
    }
}
```

### Internal Flow
1. Spring Boot creates `EntityManagerFactory`
2. `EntityManagerFactory` creates `EntityManager` instances
3. Each `EntityManager` wraps a Hibernate `Session`
4. Spring Data JPA repositories use `EntityManager`

## Answer
**Spring Data JPA uses both**: It uses `EntityManager` (JPA interface) which internally uses Hibernate's `SessionFactory` to create `Session` instances.

## Interview Tip
Explain that Spring Data JPA abstracts the JPA provider (Hibernate). You interact with `EntityManager`, but Hibernate's `SessionFactory` works behind the scenes.