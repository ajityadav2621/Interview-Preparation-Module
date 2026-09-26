# SessionFactory vs EntityManager

## SessionFactory
- **Purpose**: Factory for creating Hibernate Sessions
- **Lifecycle**: Application-level (singleton)
- **Thread-safe**: Yes, can be shared across threads
- **Configuration**: Reads Hibernate configuration

```java
// Get SessionFactory from Configuration
Configuration configuration = new Configuration().configure();
SessionFactory factory = configuration.buildSessionFactory();

// Create sessions
Session session = factory.openSession();
```

## EntityManager
- **Purpose**: JPA interface for entity lifecycle management
- **Lifecycle**: Request-level or transaction-level
- **Thread-safe**: No, must be thread-scoped
- **Provider**: Hibernate implements this interface

```java
// Get EntityManager from EntityManagerFactory
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
```

## Key Differences

| Aspect | SessionFactory | EntityManager |
|--------|---------------|---------------|
| **Interface** | Hibernate-specific | JPA standard |
| **Lifecycle** | Application singleton | Request/transaction |
| **Thread Safety** | Thread-safe | Not thread-safe |
| **Usage** | Hibernate directly | JPA abstraction |
| **Query Language** | Hibernate Query Language (HQL) | Java Persistence Query Language (JPQL) |

## Interview Tip
Explain that SessionFactory is Hibernate-specific, while EntityManager is the JPA standard. Spring Data JPA uses EntityManager internally.