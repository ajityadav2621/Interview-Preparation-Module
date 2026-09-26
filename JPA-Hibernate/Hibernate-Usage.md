# Hibernate Uses SessionFactory or EntityManager?

## Hibernate Architecture

### Hibernate Provides Both
- **SessionFactory**: Core Hibernate factory
- **EntityManager**: JPA interface implementation

### SessionFactory (Core)
```java
// Standard Hibernate usage
Configuration configuration = new Configuration().configure();
SessionFactory factory = configuration.buildSessionFactory();

// Create Session for database operations
Session session = factory.openSession();
Transaction transaction = session.beginTransaction();

// Hibernate operations using Session
User user = session.get(User.class, userId);
session.save(user);

transaction.commit();
session.close();
```

### EntityManager (JPA Standard)
```java
// JPA usage with Hibernate as provider
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
EntityTransaction transaction = em.getTransaction();

// JPA operations using EntityManager
User user = em.find(User.class, userId);
em.persist(user);

transaction.commit();
em.close();
```

## Internal Relationship
- **EntityManagerFactory** wraps Hibernate's **SessionFactory**
- **EntityManager** wraps Hibernate's **Session**
- Both provide similar functionality but different interfaces

## Interview Tip
Explain that Hibernate provides both SessionFactory (native) and EntityManager (JPA standard). Most modern applications use JPA with EntityManager.