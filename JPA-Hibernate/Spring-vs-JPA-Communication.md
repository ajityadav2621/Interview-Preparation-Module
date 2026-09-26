# Spring Communication with Database vs Spring Data JPA

## Spring Communication with Database

### Using JDBCTemplate
```java
@Service
public class UserServiceJDBC {
    
    private final JdbcTemplate jdbcTemplate;
    
    public UserServiceJDBC(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    public User findById(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?", 
        return jdbcTemplate.queryForObject(sql, new Object[]{id}, userRowMapper);
    }
    
    public void save(User user) {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        jdbcTemplate.update(sql, user.getName(), user.getEmail());
    }
}
```

### Using Hibernate Directly
```java
@Service
public class UserServiceHibernate {
    
    private final SessionFactory factory;
    
    public UserServiceHibernate(SessionFactory factory) {
        this.factory = factory;
    }
    
    public User findById(Long id) {
        Session session = factory.openSession();
        try {
            return session.get(User.class, id);
        } finally {
            session.close();
        }
    }
}
```

## Spring Data JPA Communication

### Repository Interface
```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Method name derived query
    User findByEmail(String email);
    
    // Custom query
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findByNameContaining(@Param("name") String name);
    
    // Pagination and sorting
    Page<User> findByAge(int age, Pageable pageable);
}
```

### Service Layer
```java
@Service
public class UserServiceJPA {
    
    private final UserRepository userRepository;
    
    public UserServiceJPA(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    public User save(User user) {
        return userRepository.save(user);
    }
    
    public void deleteById(Long id) {
        userRepository.delete(id);
    }
}
```

## Key Differences

| Aspect | JDBCTemplate/Hibernate | Spring Data JPA |
|--------|------------------------|-----------------|
| **Abstraction Level** | Low-level SQL/JPQL | High-level repository |
| **Code** | SQL/JPQL in code | Method names or annotations |
| **Boilerplate** | More code | Less code |
| **Learning Curve** | Steeper | Easier |
| **Flexibility** | More control | Less control |

## Interview Tip
Explain that Spring Data JPA abstracts the persistence layer, while direct JDBCTemplate/Hibernate gives more control. Choose based on team expertise and requirements.